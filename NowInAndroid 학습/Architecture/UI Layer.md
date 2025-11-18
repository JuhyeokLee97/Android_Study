## UI Layer
UI 레이어는 다음 두 요소로 구성된다.
- Jetpack Compose로 작성된 UI 컴포넌트
- Android ViewModel

ViewModel은 UseCase나 Repository로부터 데이터를 스트림(Flow) 형태로 전달받고, 이를 **UI state**로 변환한다.
UI는 이러한 상태를 기반으로 화면을 그리며 동시에 사용자 상호작용을 처리한다.</br>
사용자의 상호작용은 이벤트로 ViewModel에 전달되고, ViewModel에서 처리된다.

<image src="./docs/images/architecture-4-ui-layer.png" />

### Modeling UI state
UI state는 interface와 immutable data class를 기반으로 한 sealed 계층 구조로 모델링된다.

State 객체는 항상 데이터 스트림 변환을 통해서만 생성된다.
이러한 방법은 다음 두 가지를 보장한다.
1. UI state는 항상 앱 데이터(SSOT: Single Source Of Truth)를 반영한다.
2. UI는 sealed 구조 덕분에 가능한 모든 상태를 반드시 처리해야 한다.

#### Example: News feed on For You Screen
For You 화면의 뉴스 목록(feed)은 `NewsFeedUiState`로 표현된다.</br>
이 sealed interface는 두 가지 상태를 갖는다.
- `Loading`: 데이터 로딩 중.
- `Success`: 데이터 로딩 완료. 이 상태는 **뉴스 목록**을 포함한다.

`feedState`는 `ForYouScreen` Composable 함수로 전달되며, `ForeYouScreen`은 이 두 상태에 대해 각각 UI를 렌더링한다.

### Transforming Streams into UI state
ViewModel은 UseCase나 Repository로부터 **cold Flow**를 전달받는다.
이 여러 스트림들은 `combine` 또는 `map`을 통해서 하나의 UI state Flow로 변환된다.

이 Flow는 stateIn()을 이용해 hot Flow(StateFlow)로 변환된다.
StateFlow로 변환함으로써 UI는 "마지막으로 알려진 상태(last known state)"를 즉시 읽을 수 있게 된다.

#### Example: Displaying followed topics
`InterestsViewModel`은 `uiState`를 `StateFlow<InterestsUiState>` 형태로 외부에서 접근할 수 있도록 한다.

이 흐름은 `GetFollowableTopicsUseCase`가 제공하는 `Flow<List<FollowableTopic>>`(cold Flow)를 기반으로 만들며, 새로운 리스트가 방출될 때마다 이를 `InterestsUiState.Interests` 상태로 변환해 UI에 전달된다.

### Processing user interactions
UI 요소는 사용자의 액션을 **람다 함수 형태의 콜백**으로 ViewModel에 전달한다.
ViewModel은 이 콜백을 받아 적절한 메서드를 실행하며 이벤트를 처리한다.

#### Example: Following a topic
`InterestsScreen`은 `followTopic`이라는 람다를 파라미터로 받는다.
이 람다는 `InterestsViewModel.followTopic` 메서드를 전달받는다.

사용자가 토픽을 누르면 UI는 `followTopic()`을 호출하고, ViewModel은 이 요청을 처리하여 UserRepository에 반영한다.
