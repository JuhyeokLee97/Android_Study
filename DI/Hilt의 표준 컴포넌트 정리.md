# Android 개발을 위한 Hilt Component·Scope·Lifecycle 이해
Hilt는 Android 환경에서 최적화된 DI 프레임워크로, **Android 생명주기에 맞춰 표준 컴포넌트를 자동으로 생성**해준다.
이번 글에서는 Hilt의 표준 컴포넌트, 계층 구조, Scope, Lifecycle에 대해 학습한 내용을 정리한 것이다.

## 1. Hilt의 표준 컴포넌트란?
Dagger에서는 개발자가 직접 Component를 정의해야 하지만, Hilt는 **Android 생명주기에 대응되는 표준 Component를 자동 생성**한다.
즉, 개발자는 `@AndroidEntryPoint`와 같은 어노테이션을 마킹하면 되고,
Hilt Component 자체를 직접 정의하거나 인스턴스화 할 필요가 없다.

### Hilt 컴포넌트 계층 구조
<img src="https://developer.android.com/static/images/training/dependency-injection/hilt-hierarchy.svg"/>

Hilt의 컴포넌트는 Android 구조처럼 계층을 가진다.
컴포넌트 계층 이미지에서 볼 수 있는 각 컴포넌트 위에 있는 어노테이션은 **스코프 어노테이션**으로 해당 컴포넌트 생명주기에 대한 의존성을 지정할 때 사용한다.
화살표는 상위 컴포넌트가 하위 컴포넌트를 가르키고 있는데, 하위 컴포넌트는 상위 컴포넌트의 바인딩에 접근할 수 있다는 의미이다.
하지만 반대의 경우로 상위 컴포넌트는 하위 컴포넌트의 바인딩에 접근할 수 없다.
예를 들어 FragmentComponent에서는 ActivityComponent 바인딩에 접근하는 것은 가능하지만 ActivityComponent에서 FragmentComponent 바인딩에 접근하는 것은 불가능하다.
이는 DI 그래프의 일관성과 안정성을 보장하기 위해 Hilt가 가지는 핵심 특성이다.

#### 구체적인 예시: 상위 바인딩 접근 가능 vs 불가능
다음 예시는 이해를 돕기 위해 **Repository를 Activity 범위에 바인딩하는 경우**를 가정한다.
<strong>ActivityComponent에 바인딩한 Repository</strong>
``` kotlin
@Moudle
@InstallIn(ActivityComponent::class)
interface RepositoryModule {
    @Binds
    fun bindUserRepository(
        impl: UserRepositoryImpl
    ): UserRepository
}
```
<strong>ActivityComponent에 바인당한 Repository는 Fragment에서 사용 가능</strong>
``` kotlin
@AndroidEntryPoint
class ProfileFragment: Fragment() {
    @Inject lateinit var repository: UserRepository
}
```
FragmentComponent는 ActivityComponent의 하위이므로 ActivityComponent의 바인딩(UserRepository)을 사용할 수 있다.

<strong>반대로 Activity에서 Fragment 전용 바인딩 접근은 불가능</strong>
``` kotlin
@Module
@InstallIn(Fragmentcomponent::class)
object FragmentOnlyModule {
    @Provides
    fun porivdeFragmentHelper(): FragmentHelper = FragmentHelper()
}
```
<strong>FragmentComponent에 바인됭한 FragmentOnlyMoudle은 Activity에서 사용 불가능</strong>
``` kotlin
@AndroidEntryPoint
class MainActivity: AppCompatActivity() {
    @Inject lateinit var fragmentHelper: FragmentHelper
}
```
위와 같이 ActivityComponent의 하위 컴포넌트인 FragmentComponent의 바인딩에 접근하는 경우 컴파일 타임에 에러가 발생한다.

**Hilt 컴포넌트 계층 구조**를 정리하면 다음과 같다.
- 하위 컴포넌트는 상위 컴포넌트 바인딩에 접근할 수 있다
- 상위 컴포넌트는 하위 컴포넌트의 바인딩에 접근할 수 없다.

이 구조 덕분에 DI 그래프의 안정성이 높아지고, **컴파일 타임에 의존성 오류를 미리 잡을 수 있는 것이다.**

## 2. Hilt 컴포넌트와 Android 컴포넌트 대응
|Hilt 컴포넌트 | Android 컴포넌트|
|--|--|
|`SingletonComponent`	| `Application` |
|`ActivityRetainedComponent` |	N/A |
|`ViewModelComponent`	| `ViewModel` |
|`ActivityComponent`	| `Activity`|
|`FragmentComponent`	| `Fragment` |
|`ViewComponent`	| `View`|
|`ViewWithFragmentComponent` |	`View` annotated with `@WithFragmentBindings`|
|`ServiceComponent`	| `Service`|

## 3. Hilt 컴포넌트의 생명주기
Hilt 컴포넌트는 Android 컴포넌트의 생명주기와 일치하도록 설계되었다.
즉, Android 컴포넌트가 생성되는 시점에 맞춰 Hilt 컴포넌트도 자동 생성되고, Android 컴포넌트가 파괴될 때 Hilt 컴포넌트도 함께 파괴된다.

아래는 Hilt 표준 컴포넌트의 생명주기를 공식 문서 기준으로 정리한 표다.
|Hilt Component| Created At | Destroyed At |
|--|--|--|
| `SingleComponent` | `Application#onCreate()`| `Application` destroyed | 
| `ActivityRetainedComponent`| `Activity#onCreate()` | `Activity#onDestroy()` | 
| `ViewModelComponent` | `ViewModel` created | `ViewModel` destroyed |
| `ActivityComponent` | `Activity#onCreate()` | `Activity#onDestroy()` |
| `FragmentComponent` | `Fragment#onAttach()` | `Fragment#onDestroy()` |
| `ViewComponent` | `View#super()` | `View` destroye |
| `ViewWithFragmentComponent` | `View#super()` | `View` destroyed |
| `ServiceComponent` | `Service#onCreate()` | `Service#onDestroy()` |

## 4. 컴포넌트 Scope
Hilt에서 바인딩은 기본적으로 **Unscoped**이며, Hilt가 제공하는 Component Scope를 사용해 **특정 범위에서 재사용할 객체**를 지정할 수 있다.

|Android class|	Generated component|	Scope|
|--|--|--|
|`Application`| `SingletonComponent`| `@Singleton` |
|`Activity`| `ActivityRetainedComponent`| `@ActivityRetainedScoped` |
|`ViewModel`| `ViewModelComponent`| `@ViewModelScoped` |
|`Activity`| `ActivityComponent`| `@ActivityScoped` |
|`Fragment`| `FragmentComponent`| `@FragmentScoped` |
|`View`| `ViewComponent`| `@ViewScoped` |
|`View` annotated with `@WithFragmentBindings`| `ViewWithFragmentComponent`| `@ViewScoped` |
|`Service`| `ServiceComponent`| `@ServiceScoped` |

### Unscoped
Hilt에서 바인딩은 아무런 Scope를 지정하지 않으면 **Unscoped**이다. 그래서 **Hilt는 요청이 발생할 때마다 새로운 인스턴스를 생성**한다.

``` kotlin
class AnalyticsAdapter @Inject constructor(
  private val service: AnalyticsService
) { ... }
```
위와 같은 예시 코드의 경우, `AnalyticsAdapter`는 Activity, Fragment 등 어디서 요청하든 매번 새 인스턴스를 생성한다.
다시 말해서 하나의 Activity 내에서 2개의 `AnalyticsAdapter`를 생성하면 2개의 다른 인스턴스가 생성된다.

``` kotlin
@AndroidEntryPoint
class ExampleActivity : AppCompatActivity() {

    @Inject lateinit var adapter1: AnalyticsAdapter
    @Inject lateinit var adapter2: AnalyticsAdapter

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        Log.d("TEST", "adapter1 = $adapter1")
        Log.d("TEST", "adapter2 = $adapter2")
    }
}
```
로그를 찍어보면 `adapter1`과 `adapter2`는 서로 다른 인스턴스임을 암 수 있다.
필요할 때마다 새로 만들어도 상관 없는 가벼운 객체라면 굳이 Scope를 붙이지 않고 Unscoped로 두는 것이 자연스럽다.

### Scoped
Scope를 지정하면, Hilt는 **해당 컴포넌트 인스턴스당 한 번만 객체를 생성**하고, 그 이후로는 같은 인스턴스를 계속 반환한다.
예를 들어 `AnalyticsAdapter`를 Activity 범위에서 재사용하고 싶다면 `@ActivityScoped`를 붙이면 된다.
``` kotlin
@ActivityScoped
class AnalyticsAdapter @Inject constructor(
  private val service: AnalyticsService
) { ... }
```
위와 같이 하면 하나의 Activity 안에서 주입되는 `AnalyticsAdapter`는 **항상 같은 인스턴스**이고 Activity가 파괴되면 인스턴스토 함께 메모리에서 사라진다.


### Scope를 쓸 때 주의할 점
공식 문서에서는 **Scoped 바인딩은 Component가 파괴될 때까지 메모리에 남기 때문에 남용하면 메모리 비용이 커질 수 있는 점**을 주의하라고 한다.
그래서 Scope는 다음과 같은 경우에만 쓰느 것이 좋다.
- 내부적으로 상태를 유지해야 하는 경우
- 동기화가 필요해서 하나의 인스턴스만 있어야 하는 경우
- 생성 비용이 크고, 측정 결과 재상용이 유의미한 경우
