# LaunchedEffect, 언제 왜 사용할까?
Jetpack Compose를 사용하다보면, **suspend 함수 호출, 네비게이션, 스낵바 노출, Flow collect**처럼 UI 선언을 넘어서는 작업이 필요할 때가 있다.

하지만 Composable은 예측 불가능한 시점에, 예측 불가능한 순서로 다시 실행될 수 있는 구조다.
이런 환경에서 네트워크 요청이나 navigation 같은 외부 작업을 Composable 내부에 직접 넣으면 **중복 실행, 타이밍 오류, 취소 문제, 메모리 누수** 등의 문제가 발생할 수 있다.

그래서 Compose는 이러한 작업을 **안전하게** 수행하기 위해 **Effect API**들을 제공한다.
그중에서도 가장 많이 사용되는 것이 바로 `LaunchedEffect`다.

## Compose에서의 Side-effect란?
Compose 공식 문서에서는 **side-effect**를 다음과같이 정의한다.

> Composition 외부의 상태를 읽거나 변경하거나, 외부 시스템과의 상호작용하는 모든 작업

예를 들면 다음과 같은 것들이 모두 side-effect다.
- 네비게이션 이동
- Snackbar 표시
- 네트워크 요청
- Database 접근
- Log 출력
- suspend 함수 호출

문제는 **Composable 내부에 직접 side-effect를 처리하는 것은 매우 위험**하다.
이 문제를 해결하기 위해 사용하는 것이 **Effect API**이며, 그중에서도 가장 기본이자 핵심이 `LaunchedEffect`다.

## Effect는 무엇인가?
Effect란 UI를 직접 그리지 않은 Composable로, **Composition이 성공적으로 완료되었을 때 side-effect를 실행할 수 있도록 도와주는 API들**이다.

Effect의 핵심 개념은 다음 두 가지다.
1. Composition 생명주기에 따라 자동으로 시작/취소된다.
2. Key 값에 따라 재시작 여부가 결정된다.

이 덕분에 suspend 함수 기반의 로직을 안전하게 처리할 수 있다.

## LaunchedEffect: Composable 스코프에서 suspend 함수 실행하기
`LaunchedEffect`는 이렇게 정의된다.
``` kotlin
@Composable
@NonRestartableComposable
fun LaunchedEffect(
    key1: Any?,
    block: suspend CoroutineScope.() -> Unit
)
```

LaunchedEffect가 Composition에 들어오면 다음이 일어난다.
1. 새 Coroutine이 생성된다.
2. `block`에 작성한 suspend 함수가 실행된다.
3. 해당 LaunchedEffect가 Composition에서 사라지면 Coroutine은 **자동으로 취소**된다.
4. Key가 변경되면 **기존 Coroutine이 취소되고 새로 실행된다.**

즉, LaunchedEffect는 **Composable의 생명주기에 정확히 맞춰 작동하는 Coroutine**을 제공한다.

## 왜 LaunchedEffect를 사용해야 할까?
LaunchedEffect는 다음 두 상황에서 사용하는 것 좋은 선택이다.

### 1. Composable이 처음 실행될 때 한 번 실행해야 하는 작업
- 화면 진입 시, 네트워크 요청 1회만 실행
- UI 보여주면서 동시에 애니메이션 시작
- 화면 열리자마자 스낵바 한 번 표시
- 특정 조건이 되는 순간 네비게이션 수행

이 경우 Composable의 Recomposition과 상관없이 처음 한 번만 실행되는 안전한 환경이 필요하다.

### 2. 특정 상태값이 변경될 때마다 suspend 기반 작업을 재실행해야 하는 경우
- 설정 변경 시, 애니메이션 속도 재조정
- 키워드 변경 시, 검색 API 재호출
- 페이지 번호 변경 시, 목록 다시 요청

이런 상황에서 Key 값만 변경하면 Compose가 알아서 **Coroutine을 취소하고 재시작**해준다.

### 예시: 깜빡이는 알파 애니메이션
다음 코드는 공식 문서에 있는 예제이며, `pulseRateMs` 값이 바뀔 때마다 애니메이션 속도를 재설정한다.
``` kotlin
var pulseRateMs by remember { mutableLongStateOf(3000L) }
val alpha = remember { Animatable(1f) }

LaunchedEffect(pulseRateMs) {   // pulseRateMs 값이 변경되면 Coroutine 재시작
    while (isActive) {
        delay(pulseRateMs)
        alpha.animateTo(0f)
        alpha.animateTo(1f)
    }
}
```

이 코드가 보여주는 핵심은 다음 두 가지다.
1. **Composable이 Recomposition 되어도** Key 값이 같으면 Coroutine은 재시작되지 않는다.
2. **Key 값(`PulseRateMs`)이 바뀌면** 기존 Coroutine을 취소하고 다시 실행한다.

이 구조 덕분에 UI 상태 변경에 정확히 반응하는 애니메이션이 가능하다.

## 마무리 - LaunchedEffect는 언제, 왜 사용할까?
LaunchedEffect는 다음과 같은 상황에서 사용하는 것이 바람직하다.

### 언제
- Composable이 처음 Composition에 들어올 때 작업을 한 번 실행하고 싶을 때
- 특정 상태가 변경될 때마다 suspend 함수 기반 작업을 재실행하고 싶을 때
- UI 이벤트 기반으로 navigation, snackbar 등을 호출할 때
- Flow를 Composable에서 안전하게 collect할 때

### 왜
- Composable은 재호출되기 때문에 side-effect를 직접 넣을 수 없다.
- suspend 함수 호출은 CoroutineScope가 필요한데 LaucnhedEffect는 Coroutine을 실행하는 안전한 환경을 제공한다.