## Data Layer
Data Layer는 앱의 모든 데이터를 **offline-first 전략**으로 관리하며, 앱 전체에서 사용되는 데이터에 대한 단일 진실 공급원(SSOT, Single Source of Truth) 역할을 한다.

<img src="./docs/images/architecture-3-data-layer.png" />

각 Repository는 자신의 책임 범위에 해당하는 개별 모델을 가진다.
예를 들어 `TopicsRepository`는 `Topic` 모델을, `NewsRepository`는 `NewsResource` 모델을 갖는다.

Repository는 다른 레이어가 데이터에 접근하는 **공개 API(Public API)**로서, 앱 데이터에 접근할 수 있는 **유일한 통로**이다. 각 Repository는 일반적으로 데이터를 읽거나 쓸 수 있는 여러 메서드를 제공한다.

### Reading Data
Repository는 데이터를 스트림(Flow) 형태로 노출한다.
따라서 Repository를 사용하는 클라이언트는 데이터가 변경될 때에 **반응할 수 있어야 한다.**
데이터는 `getModel`과 같은 스냅샷 형태로 제공되지 않는다. 스냅샷은 나중에 사용할 때 이미 오래된 데이터일 수 있기 때문에 유효성을 보장할 수 없기 때문이다.

앱의 <strong>단일 진실 공급원(SSOT)</strong>은 로컬 저장소이므로 데이터 읽기 작업은 항상 로컬 저장소에서 수행된다. 이 때문에 Repository에서 데이터를 읽을 때 오류가 발생하는 경우는 거의 없다.

하지만 로컬 데이터와 원격 데이터 간의 차이를 맞추는(동기화하는) 과정에서는 네트워크 오류 등을 포함한 문제가 발생할 수 있다. 자세한 내용은 아래 "Data Synchronization" 섹션을 참고하면 된다.

#### Example: Read a list of topics
토픽 목록은 `TopicsRepository.getTopics()`가 반환하는 `Flow<List<Topic>>`을 구독함으로써 가져올 수 있다. 

토픽 목록이 변경될 때(예: 새 토픽 추가), 업데이트된 `List<Topic>`이 스트림을 통해 자동으로 emit 된다.

### Writing Data
데이터를 쓰기 위해, 그 레파지토리는 `suspend` 함수를 제공한다.
이 함수를 호출하는 곳에서는 안정된 코루틴스코프에서 호출하는 것을 확실하게 해야한다.

#### Example: Follow a topic
간략하게 `UserDataRepository.toggleFollowedTopicId`를 유저가 팔로우하고 싶은 토픽의 ID와 `followed=true` 상태를 함께 전달해서 호출하면 된다.

### Data Sources
Repository는 하나 이상의 데이터 소스를 사용한다.
예를 들어, `OfflineFirstTopicsRepository`는 다음 데이터 소스들에 의존한다.
|Name|Backed by| Purpose|
|---|---|---|
|TopicsDao|Room/SQLite| 토픽과 관련된 영구적인 관계형 데이터를 저장하고 조회하는 역할|
|NiaPreferencesDataSource|Proto Datastore|User Preferences와 관련된 비정형 데이터를 저장. 특히 사용자가 관심 있는 토픽 목록을 저장하며, 데이터 구조는 .proto 파일에서 protobuf 문법으로 정의됨 |
|NiaNetworkDataSource|Retrofit을 통한 원격 API|REST API 엔드포인트를 통해 제공되는 토픽 데이터를 JSON 형태로 가져오는 역할|

### Data Synchronization
Repository는 로컬 저장소와 원격 데이터 소스 간의 데이터를 **동기화**하는 책임을 가진다.
원격 데이터 소스에서 데이터를 가져오면 즉시 로컬 저장소에 저장한다.
이후 업데이트된 데이터는 로컬 저장소(Room)를 통해 관련 데이터 스트림으로 emit되며, 이 스트림을 구독 중인 모든 클라이언트가 최신 데이터를 전달받는다.

이 접근 방식은 읽기(Read)와 쓰기(Write) 책임을 명확히 분리하여 서로 간섭하지 않도록 보장한다.

데이터 동기화 중 오류가 발생하면, `WorkManager`의 `SyncWorker`가 exponential backoff 전략을 사용해 재시도 작업을 수행한다.

데이터 동기화 예시는 `OfflineFirstNewsRepository.syncWith`를 참고할 수 있다.