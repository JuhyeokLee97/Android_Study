# Composable의 생명주기
> 이 글은 안드로이드 공식 문서 중 CORE AREAS > UI 내용을 기반으로 학습하여 작성되었습니다.
> 
> ref). [공식 문서](https://developer.android.com/develop/ui/compose/lifecycle)

## 개요
[상태 관리 가이드](https://developer.android.com/develop/ui/compose/state)에서 언급되듯, **Composition은 Composable 함수 실행 결과로 구성되는 UI 트리 구조**를 갖고 있다.
Jetpack Compose는 Composable을 실행해 이 UI 트리를 만들어내고, 이를 기반으로 화면을 렌더링한다.

### Initial Composition
Jetpack Compose가 Composable을 **처음 실행할 때**, UI를 구성하기 위한 **초기 Composition**이 생성된다.
이 과정에서 Compose는 **어떤 Composable이 호출되었는지**, **Composable들이 어떤 UI 구조를 이루는지**를 모두 기록해 **Composition에서 사용할 트리**를 만든다.

### Recomposition
앱의 상태가 변경되면, Jetpack Compose는 Recomposition을 예약해서 변경된 상태에 영향을 받는 Composable이 다시 렌더링 되도록 한다.

참고). Composition은 초기 생성 이후에 변경/업데이트는 오직 Recomposition을 통해서만 이루어진다.

### Composition 내부에서의 Composable의 생명주기
<img src="../docs/images/lifecycle-composition.png">

- **Enter the Composition**: Composable의 생명주기는 Jetpack Compose를 통해 렌더링되고 Composition 내부의 트리에 노드 형태로 기록된다.
- **Recompose 0 or more times**: Jetpack Compose에서는 상태 변경이 감지되면 Composition 내부에 있는 Composable들 중 상태 변경에 영향을 받을 Composable을 선정해 Recomposition이 일어나도록 예약한다.
- **Leave the Composition**: Composable이 불필요해지는 경우 Composition 내부 트리에서 제거된다.


### Composition 내부 트리
동일한 Composable이 여러 번 사용(호출)되는 경우, Composition 내부에서는 각각의 객체를 기록한다.
각 호출 위치(`call site`) 별로 독립된 Composable 인스턴스가 존재하며 각각 자신의 생명주기를 가진다.

예를 들어 다음 코드를 보면
``` kotlin
@Composable
fun MyComposable() {
    Column {
        Text("Hello")
        Text("World")
    }
}
```
Composable 함수인 Column {} 내부에 동일한 Composable인 `Text()`를 2개 사용했다.
이 경우, 각각의 Text는 Composition 내부에서 고유 생명주기를 갖는다.

Composition 내부는 다음과 같이 되어 있다.
<img src="../docs/images/lifecycle-composition-hierarchy.png">


## Composition 내부에 있는 Composable 분석
Jetpack Compose에서는 Composable을 인식할 때, `call site`를 이용해서 Composable들을 구분한다.
그리고 Composition 내부의 Composable은 이 `call site`로 식별되는 것이다. 그래서 동일한 Composable을 여러 개 사용해도 구분해서 관리될 수 있는 것이다.

참고). **`call site`**: Composable 함수가 호출되는 소스 코드 위치이다.

만약 Recomposition 과정에서 한 Composable 내부에서 이전에는 호출하지 않았던 Composable 함수를 호출한다면 어떤 일이 발생할까?
Compose에서는 Composable 함수들 중 어떤 것이 호출되었고 호출되지 않은 것인지 구별하고 Composable에서 사용하는 값이 변경되지 않은 경우에는 Recomposition을 발생시키지 않는다.

Composable 내부에서 사이드이펙트를 처리하는 것을 고려할 때, 식별자를 사용하는 것은 매우 중요하다.
식별자를 사용함으로써 매번 모든 Composable에서 Recomposition이 발생하지 않도록 하고, 이 덕분에 사이드이펙트를 처리중인 Composable이 Recomposition 되지 않고 기존 작업을 완료할 수 있게 된다.

아래 예를 살펴보자.
``` kotlin
@Composable
fun LoginScreen(showError: Boolean) {
    if (showError) {
        LoginError()
    }
    LoginInput() // This call site affects where LoginInput is placed in Composition
}

@Composable
fun LoginInput() { /* ... */ }

@Composable
fun LoginError() { /* ... */ }
```
`LoginScreen`에서는 선택적으로 Composable 함수인 `LoginError`를 호출하고, `LoginInput`은 항상 호출한다.
각 Composable은 고유한 `call site`와 `source position`을 갖고 있어서 Compose 컴파일러는 식별할 수 있다.

만약 `LoginScreen`에서 Recomposition이 발생하면 `LoginInput`은 호출될 수 있지만, **Compose는 파라미터 변경이 없을 알고 실제 Recomposition 작업을 스킵**한다.
왜냐하면 `LoginInput` 함수는 Recomposition에 영향을 줄 수 있는 어떠한 파라미터도 사용하지 않기 때문이다. 그래서 Compose는 `LoginInput`의 경우 에는 스킵한다.

예를 들어 `LoginScreen`에서 사용하는 `showError` 값이 `false`에서 `true`로 바뀐다면 `LoginScreen`은 Recomposition이 발생하고 내부에 있는 Composable 함수인 `LoginError`는 렌더링되고 Composition에 노드로 추가될 것이다. 반면 `LoginInput`은 `showError`에 영향을 받지 않기 때문에 Compose에서는 Recomposition을 일으키지 않는다.

다음은 `showError` 값이 `false`에서 `true`로 바뀌면서 Composition의 `LoginScreen` 트리에 발생한 변화를 나타낸 것이다. 참고로 같은 색상의 Composable은 이전과 동일한 것이며 Recomposition이 발생하지 않았다는 의미이다.

<img src="../docs/images/lifecycle-showerror.png">
