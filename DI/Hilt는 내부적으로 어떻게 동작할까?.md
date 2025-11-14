# Hilt 내부 동작 이해하기
### Annotation Processing부터 Bytecode 변조까지
Android 개발에서 의존성 주입(DI)을 본격적으로 적용하려면 가장 먼저 만나게 되는 프레임워크가 **Hilt**다.
Hilt는 Dagger2를 기반으로 Android 환경에 최적화된 DI 솔루션이며, 복잡한 Android 컴포넌트의 생명주기(Lifecycle)를 고려한 주입 방식을 표준화한다.

이번 글은 **Hilt 내부 동작을 큰 흐름 중심으로 이해하는 것을 목표로 학습한 내용**을 전한다.
너무 심화된 내부 구현까지 들어가기보다는, "왜 이런 구조가 필요한가?"에 집중했다.

## 1. Android에서 DI가 까다로운 이유
Android 환경에서 DI가 어려운 이유는 크게 두 가지다.

### 1). Android 컴포넌트는 개발자가 직접 생성하지 않는다
Activity, Fragment 등은 Android OS가 직접 생성하며, 개발자가 생성자나 초기화 시점을 마음대로 통제할 수 없다.

### 2). 컴포넌트마다 Lifecycle이 모두 다르다
- Activity: onCreate
- Fragment: attach -> create -> view lifecycle
- Service: onCreate
- Application: Process 시작 시점

하나의 방식으로 DI를 처리할 수 없기 때문에, 각 컴포넌트의 생성 흐름을 정확히 파악하고 주입 시점을 맞춰야 한다.

Hilt는 이 문제를 **Annotation Processing + Bytecode 변조**라는 조합으로 해결한다.

## 2. Annotation & Annotation Processing
### 1). Annotation 역할
Hilt의 주요 Annotation은 다음과 같다.
- @HiltAndroidApp
- @AndroidEntryPoint
- @Module
- @InstallIn
Annotation은 **컴파일러와 Annotation Processor가 해석할 수 있는 메타데이터를 소스코드에 부착하는 기술**이다.
Hilt는 이 메타데이터를 기반으로 DI코드(Factory, Component 등)를 자동 생성한다.

### 2). Annotation Processing 이란
Hilt는 Annotation Processing(AP)을 통해 다음의 작업을 수행한다.
- 의존성 그래프 생성 및 검증
- Dagger가 사용할 Factory, Component, Module 등의 소스코드 자동 생성
- 런타임 리플렉션 없이 컴파일 타임에 필요한 DI 구조 구성

즉, Annotation Processing은 **"Hilt가 동작하기 위한 DI 코드를 컴파일 전에 미리 생성하는 단계"**라고 볼 수 있다.

## 3. Hilt의 Bytecode 변조
Dagger는 Annotation Processing만 사용한다.
하지만 **Hilt는 Android 컴포넌트에 자동으로 DI 코드를 삽입하기 위해 바이트코드 변조까지 수행**한다.

### 왜 바이트코드 변조가 필요할까?
Annotation Processing만으로는 한계가 있다.
예를 들어, 다음 작업은 Annotation Processing으로 할 수 없다.
- 이미 존재하는 Activity/Fragment의 부모 클래스 변경
- onCreate(), onAttach() 등의 생명주기 메서드 내부에 코드 삽입

그러나 DI를 자동화하려면 다음이 필요하다.
- Activity가 생성되면 inject()가 자동으로 호출되어야 함
- Fragment도 attach 시점에 inject()가 실행되어야 함
- 개발자가 일일이 상속 구조나 주입 코드를 작성하지 않아야 함

따라서 Hilt는 **컴파일된 바이트코드를 재가공**하여 Android 컴포넌트의 동작을 DI 친화적으로 변경한다.

바이트코드 변조의 구체적인 예는 다음과 같다.
- MainActivity가 실제로 Hilt_MainActivity를 부모로 상속하도록 변경
- lifecycle 내부(super.onCreate() 직전)에 inject() 호출 삽입
- 개발자가 주입 코드를 따로 작성하지 않아도 자동으로 동작

``` kotlin
@AndroidEntryPoint
class MainActivity: AppCompatActivity() 
```
겉으로 보기엔 이 코드가 전부지만, 실제로 바이트코드는 다음과 같은 구조로 변환되어 앱이 실행된다.
``` kotlin
class MainActivity: Hilt_MainActivity 
```
그리고 Hilt_MyActivity 내부에는 이미 DI 초기화 코드가 들어있다.

## 4. Hilt 내부 동작 전체 흐름
정리하면 Hilt의 전체 동작 과정은 다음과 같다. </br>
**1. 개발자가 Annotation 기반 설정** - @AndroidEntryPoint, @HiltAndroidApp, @Module 등 </br>
**2. Annotation Processing** - Dagger Compoent/Factory/Module 생성 및 의존성 그래프 검증 </br>
**3. Kotlin/Java 컴파일** - .class 바이트코드 생성 </br>
**4. Hilt 바이트코드 변조** - Android 컴포넌트의 부모 클래스를 Hilt_* 버전으로 교체 </br>
**5. D8 변환 -> Dex 생성** </br>
**6. APK/AAB 패키징** </br>
이 전체 과정이 결합되어 **Annotation만 붙여도 DI가 자동으로 작동하는** 개발 경험이 만들어진다. </br>


