# Compose에서의 Navigation
[Navigation Component](https://developer.android.com/guide/navigation)는 Jetpack Compose 앱에서도 사용할 수 있다.
이를 통해 Composable 간 화면 이동을 구현하면서 Navigation Component가 제공하는 인프라와 기능을 그대로 활용할 수 있다.

## 설정
Compose에서 Navigation을 사용하려면, app 모듈의 `build.gradle` 파일에 아래 Navigation 관련 의존성을 추가해야 한다.

``` gradle
dependencies {
    val nav_version = "2.9.6"

    implementation("androidx.navigation:navigation-compose:$nav_version")
}
```

## 시작하기
Navigation을 사용할 때, `navigation host`, `graph` 그리고 `controller`를 구현해야한다.

> 참고. Navigation에 대한 자세한 내용은 [Navigation 개요](Navigation%20개요.md) 글에서 확인할 수 있다.

## NavController 만들기
**Navigation Controller**는 내비게이션의 핵심 개념 중 하나다.
`NavController`는 내비게이션 그래프(`NavGraph`)를 사용해 그래프에 정의된 destination 간 이동을 담당한다.

Navigation Component를 사용할 때는 `NavController` 클래스를 통해 Navigation Controller를 생성한다.
그리고 `NavController`는 사용자가 방문한 destination을 추적하고, destination 간 이동을 관리하는 핵심 API다.


### Compose 에서의 NavContorller
Jetpack Compose에서 `NavController`를 만들기 위해서는 `rememberNavController()` 메서드를 사용하면 된다.

``` kotlin
val navController = rememberNavController()
```

`NavController`는 Composable 계층 구조에서 가능한 상위 계층에서 생성해야 한다.
그렇게 해야 `NavController`에 접근해야 하는 모든 Composable이 이를 참조할 수 있다.

이렇게 `NavController`를 상위에 두면, 화면 내부뿐 아니라 화면 외부 Composable들을 업데이트할 때도 `NavController`를 SSOT(Single Source of Truth)에 따라 사용할 수 있게 된다.
이는 상태 호이스팅(state hoisting)의 원칙을 따르는 방식이다.

## NavHost 만들기
`NavHost`는 현재 내비게이션 대상(destination)을 담는 UI 요소다. 사용자가 앱 안에서 화면을 이동할 때, 앱은 이 `NavHost` 안에서 다른 destination을 교체하며 화면을 전환한다.
### 1. NavHost 생성과 동시에 Graph 구성하기
Route는 특정 destination으로 이동하기 위해 필요한 정보를 포함하며, destination을 식별하는 역할을 한다.

`@Serializable` 어노테이션을 사용하면, 해당 Route 타입에 필요한 직렬화/역직렬화 로직이 자동으로 생성된다.
이 어노테이션은 Kotlin Serialization 플러그인이 제공하므로, 먼저 플러그인을 설정해야 한다.

Route를 정의한 뒤에는 `NavHost` Composable을 사용해 Navigation Graph를 구성한다.
아래 예시를 보자.
``` kotlin
@Serializable
object Profile

@Serializable
object FriendsList

val navController = rememberNavController()

NavHost(navController = navController, startDestination = Profile) {
    composable<Profile> { ProfileScreen() }
    composable<FriendsList> { FriendsListScreen() }
}
```
1. `Profile`과 `FriendsList`는 **각각 하나의 Route를 표현하는 직렬화 가능한 객체**이다.
2. `NavHost`는 `NavController`와 첫 시작인 destination에 해당하는 Route 타입을 받는다.
3. `NavHost` 람다는 내부적으로 `NavController.createGraph()`를 호출하여 `NavGraph`를 생성한다.
4. 각 `composable<T>()` 호출은 해당 Route 타입을 그래프에 destination으로 추가한다.
5. 각 `composable<>` 블록 내부에서 선언된 Composable이 `NavHost`가 해당 destination에서 렌더링할 UI를 정의한다.

### 2. NavController.createGraph()로 그래프 생성 후 NavHost에 전달하기
`NavHost` 내부에서 바로 그래프를 구성하는 방식이 Compose에서 일반적인 패턴이지만, 경우에 따라 Navigation Graph를 외부에서 먼저 생성한 뒤 `NavHost`에 전달해야 하는 상황도 있다.

이런 경우 `NavController`의 `createGraph()` 메서드를 사용하여 `NavGraph`를 직접 구성할 수 있다.

아래 코드는 앞선 예제와 동일한 그래프를 `NavHost` 외부에서 생성한 뒤, `NavHost`에 전달하는 방식이다.
``` kotlin
val navGraph by remember(navController) {
  navController.createGraph(startDestination = Profile) {
    composable<Profile> { ProfileScreen() }
    composable<FriendsList> { FriendsListScreen() }
  }
}

NavHost(navController = navController, graph = navGraph)
```

## Composable로 이동하기
Composable로 이동하려면 `NavController.navigate<T>()`를 사용해야 한다.
이 overload 버전의 `navigate()`는 하나의 `route` 인자(타입)만 받으며, 이 타입 자체가 destination을 식별하는 키 역할을 한다.

``` kotlin
@Serializable
object FriendsList

navController.navigate(route = FriendsList)
```

Navigation Graph에서 Composable로 이동하려면, 먼저 **각 destination을 하나의 타입으로 대응되도록 `NavGraph`를 정의**해야 한다.
Composable destination은 `composable<T>()`함수를 사용해 등록한다.

### Composable에서 이벤트를 외부로 전달하기
Composable 함수가 다른 화면으로 이동해야 한다고 해서 해당 Composable을 `NavController`를 직접 넘겨 `navigate()`를 호출하게 하면 안 된다. 
UDF(Unidirectional Data Flow) 원칙에 따르면, Composable은 내비게이션 로직을 직접 다루지 않고 **이벤트만 외부로 전달**해야 한다.

구체적인 예시는 아래 섹션에서 설명한다.

### 예시

``` kotlin
@Serializable
object Profile
@Serializable
object FriendsList

@Composable
fun MyAppNavHost(
    modifier: Modifier = Modifier,
    navController: NavHostController = rememberNavController(),
) {
    NavHost(
        modifier = modifier,
        navController = navController,
        startDestination = Profile
    ) {
        composable<Profile> {
            ProfileScreen(
                onNavigateToFriends = { navController.navigate(route = FriendsList) },
                /* ... */
            )
        }
        composable<FriendsList> { FriendsListScreen(/* ... */) }
    }
}

@Composable
fun ProfileScreen(
    onNavigateToFriends: () -> Unit,
    /* ... */
) {
    /* ... */
    Button(onClick = onNavigateToFriends) {
        Text(text = "See friends list")
    }
}
```

1. Navigation Graph의 각 destination은 route를 통해 생성된다. 이 route는 destionation에 필요한 정보를 나타내거나 직력화 가능한 객체 또는 클래스다.
2. `MyAppNavHost` Composable은 `NavController` 객체를 갖고 있다.
3. `navigate()` 호출은 `MyAppNavHost` 내부에서 이루어져야 하며, `ProfileScreen`처럼 UI를 선언하는 Composable에서 직접 호출하면 안 된다.
4. `ProfileScreen`에는 사용자를 `FriendsList`로 이동시키는 버튼이 있지만, 이 버튼은 직접 `navigate()`를 호출하지 않는다.
5. 대신 버튼은 `onNavigateToFriends` 파라미터로 전달된 람다 함수를 호출한다.
6. `MyAppNavHost`는 `ProfileScreen`을 `NavGraph`에 등록할 때 `onNavigateToFriends`에 `navController.navigate(route = FriendsList)`를 호출하는 람다를 전달한다.
7. 이를 통해 사용자가 `ProfileScreen`의 버튼을 눌렀을 떄 올바르게 `FriendsListScreen`으로 이동하게 된다.

## 인자와 함께 이동하기
### 인자를 갖는 Route 만들기
Destination에 특정 데이터를 전달할 필요가 있는 경우, route를 파라미터를 갖는 클래스로 정의해야 한다.
아래 예시처럼 `Profile` route는 `name` 파라미터를 갖는 data class이다.

``` kotlin
@Serializable
data class Profile(val name: String)
```

Destination에 인자를 전달할 때는 route 클래스를 인스턴스화하면서 생성자에 값을 전달하면 된다.

> Note: 인자를 전달해야 할 때는 `data class`를 사용해야 한다. 인자가 없는 경우에는 `object` 또는 `data object`를 사용해야 한다.

그리고 Nullable 한 인자를 전달해야 하는 경우에는 default 값을 설정해야 한다.

``` kotlin
@Serializable
data class Profile(val nickname: String? = null)
```

### Destination에서 type-safe한 인자 이용하기
Route 객체는 `NavBackStackEntry.toRoute()` 또는 `SavedStateHandle.toRoute()`를 통해서도 얻을 수 있다.
`composable()`를 사용해 destination을 정의하면 해당 블록에서 `NavBackStackEntry`를 파라미터로 받을 수 있다.

``` kotlin
@Serializable
data class Profile(val name: String)

val navController = rememberNavController()

NavHost(navController = navController, startDestination = Profile(name="John Smith")) {
    composable<Profile> { backStackEntry ->
        val profile: Profile = backStackEntry.toRoute()
        ProfileScreen(name = profile.name)
    }
}
```

- Navigation Graph의 `startDestination`이 `"John Smith"` 값을 갖는 `Profile` route로 설정되어 있다.
- Destination은 `composable<Profile>{}`의 블록 자체다.
- `ProfileScreen` Composable은 `name`의 값으로 `profile.name`을 사용한다.
- 그러므로 `"John Smith"` 값이 `ProfileScreen`으로 전달된다.

### 예시 코드
`NavController`와 `NavHost`를 함께 사용한 예시 코드이다.

``` kotlin
@Serializable
data class Profile(val name: String)

@Serializable
object FriendsList

// Define the ProfileScreen composable
@Composable
fun ProfileScreen(
    profile: Profile,
    onNavigateToFriendsList: () -> Unit,
) {
    Text("Profile for ${profile.name}")
    Button(onClick = { onNavigateToFriendsList() }) {
        Text("Go To Friends List")
    }
}

// Define the FriendsListScreen composable
@Composable
fun FriendsListScreen(onNavigateToProfile: () -> Unit) {
    Text("Friends List")
    Button(onClick = { onNavigateToProfile() }) {
        Text("Go to Profile")
    }
}

// Define the MyApp composable, including the `NavController` and `NavHost`.
@Composable
fun MyApp() {
    val navController = rememberNavController()
    NavHost(navController, startDestination = Profile(name = "John Smith")) {
        composable<Profile> { backStackEntry ->
            val profile: Profile = backStackEntry.toRoute()
            ProfileScreen(
                profile = profile,
                onNavigateToFriendsList = {
                    navController.navigate(route = FriendsList)
                }
            )
        }
        composable<FriendsList> {
            FriendsListScreen(
                onNavigateToProfile = {
                    navController.navigate(
                        route = Profile(name = "Aisha Devi")
                    )
                }
            )
        }
    }
}
```

위 코드에서는 Composable은 `NavController`를 직접 전달받는 대신, `NavHost`가 처리할 수 있도록 이벤트(콜백)를 외부로 노출한다.
즉, Composable은 `() -> Unit` 형태의 파라미터를 갖고 있고, 이 파라미터에는 `NavHost`로부터 `NavController.navigate()`를 호출하는 람다가 전달된다.

### 복잡한 데이터 사용하기
Navigation에서 복잡한 객체(ex. UserInfo, Product 등 전체 모델)를 직접 전달하는 것은 지양해야 한다.
대신 화면 이동 시에는 필수적인 최소 정보, 예를 들면 고유 식별자(ID)와 같은 값만 전달하는 것이 좋다.

``` kotlin
// Pass only the user ID when navigating to a new destination as argument
navController.navigate(Profile(id = "user1234"))
```

복잡한 객체는 별도의 Data Layer에서 관리하며, 이를 통해 SSOT(Single Source of Truth) 원칙을 따를 수 있다.
Destination으로 이동한 뒤에는 전달된 ID를 이용해 Data Layer에서 필요한 데이터를 조회하면 된다.

추가로 ViewModel에서 Route에 담긴 인자 값을 참조하기 위해서는 `SavedStateHandle`을 사용하면 된다.

``` kotlin
class UserViewModel(
    savedStateHandle: SavedStateHandle,
    private val userInfoRepository: UserInfoRepository
) : ViewModel() {

    private val profile = savedStateHandle.toRoute<Profile>()

    // Fetch the relevant user information from the data layer,
    // ie. userInfoRepository, based on the passed userId argument
    private val userInfo: Flow<UserInfo> = userInfoRepository.getUserInfo(profile.id)
}
```

이런 접근 방식은 Configuration Changes가 발생했을 때 데이터가 유실되는 것을 막아주고, 해당 객체가 업데이트 되거나 변경(mutation)되는 과정에서 발생할 수 있는 불일치 문제도 방지해준다.