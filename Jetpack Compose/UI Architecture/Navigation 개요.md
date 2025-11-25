# Navigation 개요
**Navigation**은 사용자가 앱 내의 다양한 콘텐츠 사이를 이동하고, 특정 화면으로 진입하거나 이전 화면으로 돌아가는 모든 상호작용을 의미한다.

Jetpack의 Navigation Component는 Navigation 라이브러리, Safe Args Gradle 플러그인, 그리고 내비게이션 구현을 지원하는 여러 도구들로 구성된다.
Navigation Component는 단순한 버튼 클릭 내비게이션부터 앱바나 내비게이션 드로어 같은 복잡한 패턴까지 다양한 내비게이션 유즈 케이스를 지원한다.

### Navigation 핵심 개념 
아래 표는 Navigation의 핵심 개념과 이를 구현할 때 사용되는 주요 타입을 정리한 것이다.

|Concept|Purpose|Type|
|--|--|--|
|Host|Host는 현재 네비게이션 대상(destination)을 담는 UI 요소다. 사용자가 앱 안에서 화면을 이동할 때, 앱은 이 Host 안에서 다른 destination을 교체하며 화면을 전환한다.| Compose: `NavHost`|
|Graph|앱 안에 있는 모든 destination이 어떤 구조로 연결되어 있는지 표현하는 데이터 구조이다. 즉, 어떤 화면에서 어떤 화면으로 이동할 수 있는지 전체 네비게이션 흐름을 정의한다.|`NavGraph`|
|Controller|네비게이션 대상(destination)들 사이의 이동을 관리하는 핵심 컨트롤러다. 화면 전환, 딥 링크 처리, 백 스택 관리 등 네비게이션에 필요한 주요 기능들을 제공한다.|`NavController`|
|Destination|네비게이션 그래프(Graph)를 구성하는 하나의 노드이며, 사용자가 이 노드로 이동하면 Host는 해당 노드에 연결된 화면(콘텐츠)을 표시한다.|`NavDestination`|
|Route|특정 대상(destination)을 고유하게 식별하며, 해당 대상에 전달할 데이터도 함께 포함한다. Navigation에서는 Route를 사용해 원하는 대상으로 이동하고, 필요한 데이터를 함께 전달할 수 있다.|Any serializable data type|

<img src="../docs/images/navigation_structure.png">

### Navigation 장점 및 특징
- **애니메이션 그리고 화면 전환**: 화면 전환 시 사용할 수 있는 기본 애니메이션과 전환 리소스를 제공해, 자연스러운 이동 경험을 쉽게 구현할 수 있다.
- **딥 링킹**: 외부 링크나 알림 등을 통해 앱의 특정 화면으로 바로 이동하는 "딥 링크" 기능을 간단하게 처리할 수 있다.
- **UI 패턴**: 내비게이션 드로워, 바텀 내비게이션 같은 자주 사용되는 UI 패턴을 최소한의 코드만으로 적용할 수 있다. 복잡한 화면 구조라도 Navigation을 사용하면 관리가 수월해진다.
- **타입 안정성**: 화면 간 데이터를 전달할 때 타입이 보장되도록 설계되어 있어, 실행 중 발생하는 오류나 잘못된 파라미터 전달을 줄일 수 있다.
- **ViewModel 지원**: 내비게이션 그래프 단위로 ViewModel을 스코프(scope)로 둘 수 있어서, 여러 화면에서 동일한 UI 데이터를 쉽게 공유할 수 있다.
- **Fragment 전환**: Fragment 기반 UI에서는 각종 Fragment 트랜잭션(add, repalce, back stack 관리 등)을 Navigation이 알아서 처리해준다.
- **Back and up**: 기본적인 Back 버튼 및 상단 Up 버튼 동작을 정확하게 처리해준다.