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
더 구체적인 예로는 ...(뭐라고 설명하지...? 구체적인 예시 코드가 있으면 좋을텐데)...

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
