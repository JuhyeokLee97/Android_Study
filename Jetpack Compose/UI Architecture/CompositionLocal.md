# CompositionLocal로 구성하는 로컬 데이터 스코프
`CompositionLocal`은 Composition 트리의 하위 계층으로 데이터를 **암묵적으로** 전달하기 위한 도구다.
이 글에서는 [공식 문서](https://developer.android.com/develop/ui/compose/compositionlocal)를 바탕으로 `CompositionLocal`이 무엇인지 조금 더 자세히 살펴보고, 어떻게 정의하고 사용하는지, 그리고 어떤 시나리오에서 `CompositionLocal`을 솔루션으로 선택할 수 있을지 정리한다.

## `CompositionLocal` 개요
Compose에서 데이터는 일반적으로 UI 트리 아래로 **파라미터를 통해 명시적으로 전달**된다. 이 방식은 각 Composable의 의존성을 명확히 드러낸다는 장점이 있다.

하지만 색상, 타이포그래피, 스타일처럼 **앱 전반에서 자주 사용되는 데이터**를 모든 하위 Composable에 일일이 전달하는 것은 매우 번거롭다.

아래 예시를 살펴보자.
``` kotlin
@Composable
fun MyApp() {
    // Theme information tends to be defined near the root of the application
    val colors = colors()
}

// Some composable deep in the hierarchy
@Composable
fun SomeTextLabel(labelText: String) {
    Text(
        text = labelText,
        color = colors.onPrimary // <- need to access colors here
    )
}
```

대부분의 Composable에 colors를 **명시적인 파라미터 의존성**으로 전달하지 않아도 되도록, Compose에서는 `CompositionLocal`을 제공한다.
**`CompositionLocal`을 사용하면 트리 범위(tree-scoped) 내에서 이름이 있는 객체를 정의할 수 있으며, 이를 통해 데이터가 UI 트리를 따라 암묵적으로 전달되도록 만들 수 있다.**