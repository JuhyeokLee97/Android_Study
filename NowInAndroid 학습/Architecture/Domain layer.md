## Domain layer
[Domain layer](https://developer.android.com/topic/architecture/domain-layer)는 UseCase들을 포함한다.
각 UseCase는 비즈니스 로직을 갖고 있으며, operator fun invoke 형태의 단일 실행 메서드를 가진 클래스다.

UseCase는 ViewModel에 중복된 로직이 생기는 것을 방지하고, 공통 비즈니스 로직을 분리해 단순화하기 위해 사용된다.
주로 여러 Repository에서 가져온 데이터를 결합하거나 변화하는 역할을 한다.

예를 들어, `GetUserNewsResourcesUseCase`는 `NewsRepository`가 제공하는 `NewsResources` 스트림과 `UserDataRepository`가 제공하는 `UserData` 스트림을 `Flow`로 결합해 `UserNewsResources` 스트림을 만들어낸다.

각 ViewModel은 이 스트림을 사용하여 "뉴스 + 북마크 여부"가 포함된 형태로 화면을 렌더링한다.

특히 NIA의 Domain layer는 현재 기준으로 이벤트 처리용 UseCase를 포함하지 않는다.
이벤트는 UI layer가 직접 Repository의 메서드를 호출하는 방식으로 처리한다.