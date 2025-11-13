# Hilt는 왜 등장했을까? Android 개발자를 위한 DI 표준 솔루션 이해하기

Android 개발에서 의존성 주입(Dependency Injection, DI)은 규모가 커질수록 반드시 필요해지는 기술이다.

하지만 DI를 구현하는 기존 솔루션들은 Android 환경에서 여러 문제들이 있었고, 이 문제들을 해결하기 위해 Google이 만든 Android 전용 DI 솔루션이 바로 Hilt이다.

이번 글에서는 **Hilt가 왜 등장했는지**, 그리고 왜 **Android 개발자가 Hilt를 사용해야 하는지** 정리해본다.

## 1. Hilt가 등장한 배경
Android 개발자들이 처음부터 Hilt를 사용할 수 있었던 것은 아니다.
DI 발전 과정은 아래와 같이 흘러왔다.

### 1). Guice - 서버 환경 기반의 리플렉션 DI
Guice는 Java 진영의 DI 도구이지만 **리플렉션 기반 런타임 주입**을 사용한다.
그래서 클래스/필드를 런타임에 탐색하고 **런타임 비용이 증가**하는 단점이 있다.
Android 관점에서 보면 DI 오류를 런타임에서 확인할 수 있어서 크래시 위험성을 가져야한다는 치명적인 단점이 있다.
또한 서버 환경에서는 문제 없을 수 있지만, **리소스가 제한된 Android에서는 앱 성능이 감소**되어 비효율적이다.

즉, Android 앱에는 맞지 않았다.

### 2). Dagger2 - 성능은 좋지만 설정이 너무 복잡함
Dagger2는 Guice의 문제를 해결했다.
**리플렉션을 사용하지 않고, 컴파일 타임 코드 생성 기반**으로 DI 오류를 컴파일 단계에서 잡을 수 있다.
하지만 설정이 어렵고 학습 난이도가 높으며 프로젝트마다 사용하는 컴포넌트 구조가 달라서 표준화가 어렵다.

"성능은 좋지만 너무 어렵다"라는 비판이 많았다.

### 3). Koin - 간단하고 편하지만, 안정성에 한계
Koin은 Kotlin DSL 기반이라 사용하기 편하다.
```kotlin
startKoin {
    modules(appModule)
}
```
하지만 내부적으로 리플렉션 기반 런타입 주입이어서 컴파일 시점에 오류를 체크할 수 없다는 단점과 앱 성능 저하의 단점을 가지고 있다.

Koin은 편하지만 대형 Android 프로젝트에서 사용하기에는 최선의 선택이 아니었다.

### 4). 결국 Google이 만든 해결책: Hilt
Google은 Android 생태계에 적합한 DI 솔루션이 필요하다는 사실을 깨닫고, Dagger2를 기반으로 Android 전용 레이어를 추가한 **Hilt**를 개발했다.

Hilt는 아랭 항목을 모두 충족한다.
- Dagger2의 성능 & 안정성
- Koin처럼 간단한 사용성
- Android 컴포넌트(Activity, Fragment, ViewModel 등)용 표준 컴포넌트 제공
- Jetpack과의 통합
- 테스트 환경 지원

즉, **성능은 Dagger, 사용성은 Koin, 안정성은 Android 공식 표준** 이렇게 3가지를 모두 만족시키는 솔루션이 Hilt다.

## 2. 왜 Android 개발자는 Hilt를 사용해야 할까?
### 1). Android에 최적화된 DI 표준 솔루션
Hilt는 Android 컴포넌트를 위한 표준화된 DI범위(Component)를 제공한다.
- SingletonComponent
- ActivityRetainedComponent
- ViewModelComponent
- ActivityComponet 등
즉, "어떤 객체를 어디 범위에 넣어야 할까?"에 대한 가이드를 Hilt가 이미 제공한다.

또한 Jetpack과 완전 통합되어 있다.
- @HiltViewModel
- HiltWorkerFactory
- Navigation Graph와 주입 연동
이런 표준화된 방식 덕분에 팀 간 DI 사용 방식이 일관될 수 있다.

### 2). 컴파일 타임 코드 생성 -> 성능 & 안정성 최고 수준
Hilt는 Dagger2처럼 Annotation Processor 기반으로 동작한다.
- 리플렉션 완전 제거
- DI 오류를 **컴파일 시점**에 발견
- 런타임 안정성 극대화
실제로 "DI 오류로 앱이 런타임에 죽는 상황"을 거의 없애준다.

### 3). 보일러플레이트 최소화
Dagger2에서 필요했던 복잡한 Component/Module 정의를 Hilt는 단 몇 개의 어노테이션으로 해결한다.
- @AndroidEntryPoint
- @InstallIn
- @HiltViewModel
덕분에 Dagger2처럼 많은 설정 파일들을 만들 필요가 없다.

### 4). 테스트 환경 구성도 쉬움
Android 테스트에서는 DI 환경을 따로 구성해야 한다.

Hilt는 이를 위해 **테스트 전용 컴포넌트**와 @TestInstallIn API를 제공한다.
덕분에 실제 코드 수정 없이 테스트 DI 환경 구성이 가능하고 Mock/Stub 교체가 쉬워졌다.

## 마무리
Hilt는 2020년도에 출시되어 현재는 당연히 Android DI 솔루션으로는 Hilt를 사용할 것이다.
하지만 솔루션을 사용할 때, 최신 트렌드이니까 사용하는 것과 왜 모두가 사용하는지 알고 사용하는 것은 깊이가 다르다.
"왜 이 기술을 사용하지?"라는 의문을 해소하게 되면 자연스럽게 기술의 장점을 알게 되고 그 장점을 효과적으로 사용할 수 있다.
Android 앱을 개발한다면 "DI를 도입할까?"를 넘어서 "DI를 해야 하는데, 그중 최선의 솔루션은 무엇일까?"에 대한 답으로 Hilt가 왜 답인지 알 수 있는 학습이였다.










