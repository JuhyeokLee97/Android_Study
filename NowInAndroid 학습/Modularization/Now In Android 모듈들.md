## Now In Android 모듈들
Now In Android 앱은 위에 설명된 모듈화 전략을 사용해서 다음과 같은 모듈들을 사용한다.

|Name|Responsibilities|Key classes and good examples|
|--|--|--|
|`app`|앱이 정상적으로 동작하는 데 필요한 요소들을 모두 한데 모아 구성하는 역할을 한다. 여기에는 UI 스캐폴딩과 내비게이션과 같은 요소도 포함된다.|`NiaApp`, `MainActivity`, `NiaNavHost`, `NiaAppState`, `TopLevelDestination`|
|`feature:1`, `feature:2`, ...|특정 사용자 여정(user journey)이나 기능 단위를 담당하는 역할을 한다. 보통 이 모듈 안에는 UI를 구성하는 컴포넌트들과 다른 모듈에서 제공하는 데이터를 읽어 화면 상태를 만드는 ViewModel이 포함된다. <br/> 예를 들면 `feature:topic` 모듈은 `TopicScreen`에서 특정 토픽 정보를 화면에 보여주는 기능을 담당한다.<br/>또 다른 예로 `feature:foryou` 모듈은 `ForYouScreen`에서 첫 실행 시 온보딩 흐름을 제공하고 동시에 사용자의 뉴스 피드를 보여주는 역할을 한다.| `TopicScreen`, `TopicViewModel`|
|`core:data`|여러 데이터 소스로부터 앱 데이터를 가져와, 이를 다양한 feature에서 공통으로 사용할 수 있도록 제공하는 역할을 한다.|`TopicRepository`|
|`core:designsystem`|디자인 시스템은 Core UI 컴포넌트들(상당 수는 Material 3 컴포넌트를 커스터마이징한 것들), 앱 테마, 아이콘으로 구성된다. 이 디자인 시스템은 `app-nia-catalog` 실행 구성을 통해 직접 확인할 수 있다.|`NiaIcons`, `NiaButton`, `NiaTheme`|
|`core:ui`| 뉴스 피드처럼 여러 feature 모듈에서 공통으로 사용하는 UI 컴포넌트와 리소스를 제공한다. `designsystem` 모듈과 달리, 이 모듈은 데이터 레이어에 의존하는데 그 이유는 뉴스 리소스 같은 데이터 모델을 화면에 직접 렌더링하기 때문이다.|`NewsFeed`, `NewsResourceCardExpanded`|
|`core:common`|여러 모듈에서 공통으로 사용되는 클래스들을 포함한다.|`NiaDispatchers`, `Result`|
|`core:network`| 원격 데이터 소스에 네트워크 요청을 보내고, 그에 대한 응답을 처리하는 역할을 담당한다.|`RetrofitNiaNetworkApi`|
|`core:testing`|테스트에 사용되는 의존성, 레포지토리, 유틸 클래스들을 포함한다.|`NiaTestRunner`, `TestDispatcherRule`|
|`core:datastore`|`DataStore`를 사용해 앱에서 지속적으로 유지해야 하는 데이터를 저장한다.|`NiaPreferences`, `UserPreferencesSerializer`|
|`core:database`|`Room`을 사용해 로컬 데이터베이스에 데이터를 저장한다.|`NiaDatabase`, `DatabaseMigrations`, `Dao` 클래스들|
|`core:model`|앱 전반에서 사용되는 모델 클래스들을 포함한다.|`Topic`, `Episode`, `NewsResource`|
