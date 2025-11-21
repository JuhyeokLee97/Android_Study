# Compose에서의 Side-effect
**Side-effect**란 **Composable 함수 스코프를 벗어난 곳에서 앱 상태를 변경하는 행위**를 의미한다.
Composable은 생명주기 특성상 **예측 불가능한 Recomposition**이 발생할 수 있고, **Composable들의 실행 순서가 보장되지 않고**, **Recomposition 과정 중에 취소**될 수 있기 때문에, Composable 함수는 side-effect 없는 방향으로 개발하는 것이 이상적이다.

</br>
하지만 side-effect가 필요한 경우도 있다.
예를 들면 **스낵바를 한 번만 표시해야 한다거나, 다음 화면으로 이동해야하는 상황**과 같이 한 번만 실행되어야 하는 이벤트들이 필요할 수 있다.

이처럼 한 번만 실행되어야 하는 이벤트를 처리하기 위해서는 **Composable의 생명주기를 인지하고 있는 컨트롤 가능한 환경에서 처리해야 한다.**

이 글은 Jetpack Compose가 제공하는 다양한 **side-effect API**들에 대해 이야기한다.

## State and effect use cases
[Thinking in Compose](https://developer.android.com/develop/ui/compose/mental-model) 공식 문서에서 설명되어 있듯이, Composable은 side-effect가 없는 방향으로 개발하는 것이 이상적이다.
하지만 상황에 따라 UI 선언 외에 앱 상태를 변경하는 effect를 처리해야 하는 경우가 발생한다면, side-effect를 안정적으로 처리하기 위해서 **Effect API**들을 사용해야 한다.

> 여기서 말하는 Effect란, UI를 직접적으로 출력하지 않는 Composable 함수로, Composition이 완료된 뒤 side-effect를 실행시키는 역할을 한다.

</br>
Compose에서 Effect는 다양한 가능성을 제공하지만, 그만큼 **과도하게 사용되기 쉽다.**
Effect 내부에서 수행하는 작업이 반드시 UI 관련 작업인지 확인하고, [Managing State](https://developer.android.com/develop/ui/compose/state#unidirectional-data-flow-in-jetpack-compose) 문서에서 설명하듯, 단방향 데이터 흐름(UDF)을 해치지 않도록 주의해야 한다.

### `LaunchedEffect`: Composable의 스코프에서 suspend 함수 실행하기
Composable의 생명주기 동안 특정 작업을 수행해야 하고, 그 과정에서 suspend 함수를 호출해야 한다면 `LaunchedEffect`를 사용하면 된다.
LaunchedEffect가 Composition에 진입하면, 파라미터로 전달된 코드 블록을 실행하는 **Coroutine이 새로 시작**된다.
그리고 LaunchedEffect가 Composition에서 제거되면 해당 **Coroutine은 자동으로 취소된다.**

만약 LaunchedEffect에 설정된 key가 변경되면 Recomposition이 발생하며 **기존 Coroutine은 취소되고, 새로운 suspend 함수 블록이 새 Coroutine으로 다시 실행**된다.

예를 들어, 다음 코드는 지연 시간을 설정값으로 받아 알파 값을 깜빡이도록 만드는 애니메이션이다.
``` kotlin
// Allow the pulse rate to be configured, so it can be sped up if the user is running out of time
var pulseRateMs by remember { mutableLongStateOf(3000L) }
val alpha = remember { Animatable(1f) }
LaunchedEffect(pulseRateMs) {   // Restart the effect when the pulse rate changes
    while (isActive) {
        delay(pulseRateMs)  // Pulse the alpha evey pulseRateMs to alert the user
        alpha.animateTo(0f)
        alpha.animateTo(1f)
    }
}
```
위 코드에서 애니메이션은 delay suspend 함수를 사용해 지정된 시간만큼 대기하고, 그 후 `animateTo`를 사용해 알파 값을 0으로, 다시 1로 순차적으로 애니메이션한다. 이 동작은 Composable이 Composition에 존재하는 동안 계속 반복된다.

### `rememberCoroutineScope`: Composable 외부에서 coroutine을 실행하되, composition-aware scope 확보하기
LaunchedEffect는 **Composable 내부에서만** 사용할 수 있다.
만약 **onClick 같은 이벤트 콜백 등, Composable이 실행되는 시점이 아닌 곳에서** coroutine을 실행해야 하지만, 그 coroutine의 생명주기를 여전히 해당 Composable의 Composition에 묶고 싶다면, 다시 말해서 해당 Composable이 사라질 때 자동으로 취소되도록 만들고 싶다면 `rememberCoroutineScope`를 사용하면 된다.

또한 하나 이상의 coroutine의 생명주기를 직접 제어해야 하는 상황, 예를 들어 사용자의 특정 이벤트가 발생했을 때 애니메이션을 취소해야 하는 경우에도 `rememberCoroutineScope` 사용이 적합하다.

</br>
`rememberCoroutineScope`는 Composable 함수이며, 호출된 Composition 지점에 바인딩된 **CoroutineScope**를 반환한다.
이 Scope는 해당 호출이 Composition에서 사라질 때 **자동으로 취소**된다.

아래 코드는 사용자가 Button을 눌렀을 때 Snackbar를 표시하는 것이다.
``` kotlin
@Composable
fun MoviesScreen(snackbarHostState: SnackbarHostState) {

    // Creates a CoroutineScope bound to the MoviesScreen's lifecycle
    val scope = rememberCoroutineScope()

    Scaffold(
        snackbarHost = {
            SnackbarHost(hostState = snackbarHostState)
        }
    ) { contentPadding ->
        Column(Modifier.padding(contentPadding)) {
            Button(
                onClick = {
                    // Create a new coroutine in the event handler to show a snackbar
                    scope.launch {
                        snackbarHostState.showSnackbar("Something happened!")
                    }
                }
            ) {
                Text("Press Me")
            }
        }

    }
}
```
위 코드에서 `rememberCoroutineScope()`는 MoviesScreen Composable의 Composition 생명주기에 연결된 CoroutineScope를 생성한다.
그리고 Button 클릭 이벤트 핸들러 내부(**Composable이 실제로 실행되는 시점이 아닌 곳**)에서 scope.launch를 사용하여 Snackbar를 출력하는 coroutine을 실행한다.

### `DisposableEffect`: 정리가 필요한 side-effect 처리하기
Composable이 Composition에서 사라지거나, Key가 변경되었을 때 **정리**가 필요한 side-effect가 있다면 `DisposableEffect`를 사용하면 된다. `DisposableEffect`의 Key가 변경되면, 현재 effect를 먼저 **정리(`dispose`)**한 뒤, 새 Key에 맞춰 effect를 **다시 실행**한다.

예를 들어 `LifecycleObserver`를 사용하여 Lifecycle 이벤트에 따라 analytics 이벤트를 전송하고 싶다면, Compose에서는 **필요한 시점에 observer를 등록하고 제거하기 위해** `DisposableEffect`를 사용할 수 있다.
``` kotlin
@Composable
fun HomeScreen(
    lifecycleOwner: LifecycleOwner = LocalLifecycleOwner.current,
    onStart: () -> Unit,    // Send the 'started' analytics event
    onStop: () -> Unit      // Send the 'stopped' analytics event
) {
    // Safely update the current lambdas when a new one is provided
    val currentOnStart by rememberUpdatedState(onStart)
    val currentOnStop by rememberUpdatedState(onStop)

    // If `lifecycleOwner` changes, dispose and reset the effect
    DisposableEffect(lifecycleOwner) {
        // Create an observer that triggers our remembered callbacks for sending analytics events
        val observer = LifecycleEventObserver { _, event -> 
            if (event == Lifecycle.Event.ON_START) {
                currentOnStart()
            } else if (event == Lifecycle.Event.ON_STOP) {
                currentOnStop()
            }
        }

        // Add the observer to the lifecycle
        lifecycleOwner.lifecycle.addObserver(observer)

        // When the effect leaves the Composition, remove the observer
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }

    /* Home screen content */
}
```
위 코드에서 effect는 lifecycleOwner에 observer를 등록한다.
그리고 lifecycleOwner가 변경되면 **기존 effect는 dispose되고, 새로운 lifecycleOwner로 다시 effect가 실행된다.**

`DisposableEffect`는 블록의 마지막 구문으로는 **반드시 `onDispose` 절을 포함해야 한다.**
그렇지 않으면 IDE가 빌드 시 에러를 표시한다.

> Note: `onDispose` 내부가 비어 있는 것은 좋은 사용법이 아니다. 그런 경우 더 적절한 Effect를 사용할 수 있는지 고려해봐야 한다.

### `SideEffect`: Compose state를 Compose 외부 코드에 전달하기
Compose에서 관리하지 않는 객체와 Compose state를 공유하려면 `SideEffect` composable을 사용하면 된다. 그리고 `SideEffect`는 Composable의 Initial Composition 또는 Recomposition이 성공적으로 끝난 직후 실행되는 것을 보장한다.

반면, side-effect를 Composable 본문에 직접 호출하면 **Composition이 성공적으로 끝나기 전에 effect가 실행될 위험이 있어 올바르지 않는 방식**이다.

예를 들어 Analytics 라이브러리가 사용자에게 커스텀 메타데이터(아래 예시에서는 `user.userType` 파라미터)를 설정해 이후 발생하는 모든 analytics 이벤트에 해당 정보가 포함되도록 한다고 하자.
현재 사용자의 `userType`을 analytics 라이브러리에 전달하려면, `SideEffect`를 사용하여 해당 값을 업데이트하면 된다.

``` kotlin
@Composable
fun rememberFirebaseAnalytics(user: User): FirebaseAnalytics {
    val analytics: FirebaseAnalytics = remember {
        FirebaseAnalytics()
    }
    // On every successful composition, update FirebaseAnalytics with
    // the userType from the current User, ensuring that future analytics
    // events have this metadata attached
    SideEffect {
        analytics.setUserProperty("userType", user.userType)
    }
    return analytics
}
```

### `produceState`: non-Compose state를 Compose state로 변환하기
`produceState`는 **Composition 내에서 스코프된 coroutine을 실행하여** 그 coroutine 내부에서 값을 방출하고, 그 결과를 `State` 형태로 반환하는 함수다.
`produceState`를 사용하면 `Flow`, `LiveData`, `RxJava` 같은 **Compose가 직접 관리하지 않는 외부의 구독 기반 상태(non-Compose state)를 Compose state**로 변환할 수 있다.

`produceState`의 producer 블록은 해당 Composable이 Composition에 진입할 때 실행되며, Composition에서 벗어날 때 자동을 취소된다.
또한 반환된 State는 **conflate** 동작을 하므로 같은 값을 반복해서 설정해도 Recomposition은 발생하지 않는다.

`produceState`는 내부에서 Coroutine을 생성하지만, **suspend 함수가 아닌 일반 콜백 기반 데이터 소스**를 관찰(구독)하는 용도로도 사용할 수 있다.
이 경우, 해당 데이터 소스에서 구독을 해제하려면 `awaitDispose`를 사용하면 된다.

아례 예시는 `produceState`를 사용해 네트워크 이미지를 로드하는 방법을 보여준다.
`loadNetworkImage` Composable 함수는 다른 Composable에서 사용할 수 있는 `State`를 반환한다.
``` kotlin
@Composable
fun loadNetworkImage(
    url: String,
    imageRepository: ImageRepository = ImageRepository()
): State<Result<Image>> {
    // Creates a State<T> with Result.Loading as initial value
    // If either `url` or `imageRepository` changes, the running producer
    // will cancel and will be re-launched with the new inputs.
    return produceState<Result<Image>>(initialValue = Result.Loading, url, imageRepository) {
        // In a coroutine, can make suspend calls
        val image = imageRespository.load(url)

        // Update State with either an Error or Success result.
        // This will trigger a recomposition where this State is read
        value = if (image == null) {
            Result.Error
        } else {
            Result.Success(image)
        }
    }
}
```

> **참고**: 반환 타입이 있는 Composable 함수는 일반 Kotlin 함수처럼 소문자로 시작하는 이름을 사용하는 것이 권장된다.

> 핵심 내용: `produceState`는 내부적ㅇ로 다른 Effect API들을 이용해 동작한다. 결과값은 `remember { mutableStateOf(initialValue) }`로 저장하며, producer 블록은 LanchedEffect 안에서 실행된다.
> producer 블록에서 value가 갱신될 대마다, 이 `state`도 새로운 값으로 업데이트된다.
>
> 이렇게 Compose는 기존 Effect API를 조합하여 새로운 Effect를 만드는 것을 쉽게 할 수 있다.
