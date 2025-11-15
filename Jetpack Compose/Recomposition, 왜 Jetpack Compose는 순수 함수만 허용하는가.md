# 왜 Jetpack Compose는 순수 함수만 허용하는가: Recomposition 원리와 안전한 코드 작성법
> 이 글은 안드로이드 공식 문서 중 CORE AREAS > UI 내용을 기반으로 작성되었습니다.
> 
> ref). [공식 문서](https://developer.android.com/develop/ui/compose/mental-model)

Jetpack Compose는 선언형 UI(Declarative UI) 패러다임을 기반으로 하며, 기존 Android View 시스템에서 사용하던 명령형 UI(Imperative UI) 방식과 완전히 다른 사고방식을 요구한다.
선언형 UI의 핵심은 "데이터가 바뀌면 UI를 다시 그리는 것"이며, Compose에서는 이를 **Recomposition**이라고 부른다.

Recomposition을 올바르게 이해하기 위해서는 다음과 같은 Compose의 특성과 미래 방향성까지 함께 고려해야 한다.
1. Recomposition은 필요한 부분만 다시 실행한다
2. Recomposition은 추정 기반 재구성이며 언제든지 취소될 수 있다
3. Composable은 매우 자주 실행될 수 있다
4. Composable은 미래에 병렬로 실행될 수 있다
5. Composable은 미래에 어떤 순서로 실행될지 보장되지 않는다

따라서 **모든 Composable은 빠르고 idempotent이며, side-effect가 없어야 한다.**

## 1. Recomposition이란?
명령형 UI에서는 View의 속성을 setter로 변경해 UI를 업데이트했다. 반면 Compose에서는 **데이터가 바뀌면 Composable을 다시 호출**하여 UI를 갱신하고 이를 **Recomposition**이라 한다.
``` kotlin
@Composable
fun ClickCounter(clicks: Int, onClick: () -> Unit) {
    Button(onclick = onClick) {
        Text("I've been clicked $clicks times")
    }
}
```
`clicks`가 바뀌면 Compose는 이 Composable을 재실행(Recomposition)하여 새로운 UI로 업데이트한다.

> 참고, Compose는 의존성이 있는 Composable만 다시 실행하여 효율적으로 UI를 업데이트한다.

## 2. Recomposition은 필요한 부분만 다시 실행한다
Compose에서 Recomposition은 전체 UI를 다시 실행하는 방식이 아니다. 입력값이 바뀌었을 때, 그 값에 직접적으로 의존하는 Composable만 선택적으로 다시 실행된다. 
이를 통해 Compose는 불필요한 계산을 피하고, 화면 전체를 다시 그리는 데 드는 비용을 최소화한다.

이 개념을 잘 보여주는 예제가 아래 코드다. 이 UI는 `header`와 `names`라는 두 가지 독립적인 데이터에 의존하는 두 영역으로 구성된다.
상단의 Header(Text)는 `header` 값에만 의존하고, 리스트 아이템은 `names[i]` 각각에 의존한다.

``` kotlin
/**
 * Display a list of names the user can click with a header
 */
@Composable
fun NamePicker(
    header: String,
    names: List<String>,
    onNameClicked: (String) -> Unit
) {
    Column {
        // this will recompose when [header] changes, but not when [names] changes
        Text(header, style = MaterialTheme.typography.bodyLarge)
        HorizontalDivider()

        LazyColumn {
            items(names) { name ->
                // When an item's [name] updates, the adapter for that item
                // will recompose. This will not recompose when [header] changes
                NamePickerItem(name, onNameClicked)
            }
        }
    }
}

/**
 * Display a single name the user can click.
 */
@Composable
private fun NamePickerItem(name: String, onClicked: (String) -> Unit) {
    Text(name, Modifier.clickable(onClick = { onClicked(name) }))
}
```
이 예제의 핵심은 다음과 같다.
- `header`가 변경되면 `Text(header)`만 다시 실행되고, 리스트는 영향을 받지 않는다
- `names[i]` 중 일부가 변경되면 그 아이템만 다시 실행되며, Header나 다른 아이템은 재구성되지 않는다.
- LazyColumn 전체가 다시 실행되는 것도 아니며, Column이나 그 부모 Composable도 필요하지 않다면 건너뛰어진다

Compose는 내부적으로 UI 트리를 기억하고 있고, 각 Composable이 어떤 입력에 의존하는지 추적한다.
이러한 정보가 있기 때문에 변경된 부분만 정확히 재실행하여 성능을 최적화할 수 있는 것이다.
즉, Composable의 핵심 장점 중 하나는 **필요한 부분만 정교하게 다시 그리는 능력(Recomposition)**이다.

## 3. Recomposition은 추정 기반 재구성이며 언제든 취소될 수 있다
Compose는 "입력이 바뀌었을 가능성이 있다"고 **추정**하는 순간 Recomposition을 시작한다.
하지만 Recomposition이 끝나기 전에 또 입력이 바뀌면 **진행 중이던 Recomposition을 취소하고, 새로운 값으로 Recomposition을 다시 시작한다.**
취소된 Recomposition 내부에서 실행된 side-effect는 되돌릴 수 없다. **UI는 변경되지 않았는데, 내부 상태는 바뀌는 불일치 문제가 발생할 수 있는 것이다.**

즉, **Composable 내부에서 ViewModel 업데이트, 전역 상태 변경 등 side-effect를 절대 발생시키면 안되는 이유다.**

## 4. Composable은 매우 자주 실행될 수 있다
애니메이션이 포함된 UI는 1초에 수십~수백 번 Recomposition될 수 있다.
따라서 다음 작업은 Composable 내부에서 동작하게 하는 것은 금지된다.
- SharedPreferences 읽기
- DB 쿼리
- 파일 IO
- 네트워크 호출
- 무거운 계산

이러한 작업은 반드시 ViewModel 또는 Background coroutine에서 수행하고, 결과만 Composable로 전달해야 한다.

## 5. Composable은 미래에 병롤로 실행될 수 있다
Compose는 현재 단일 메인 스레으데서 실행되지만, 공식 문서에서 명시한 것처럼 **\"미래에는 Composable이 병렬로 실행될 수 있다\"**는 점이 매우 중요하다.
이는 단순한 가능성이 아니라 Compose 설계 철학의 핵심이며, 개발자가 지금부터 이 설계 방향을 고려해 코드를 작성해야 한다.

Compose가 병렬 Recomposition을 지원하게 되면 다음과 같은 최적화가 가능해진다.
1. 여러 Composable을 **서로 다른 CPU 코어**에서 동시에 실행
2. **화면에 보이지 않는 요소**는 낮은 우선순위 스레드에서 재구성
3. 전체 UI를 더 빠르게 계산하기 위한 **백그라운드 스레드 기반 재구성**

하지만 이러한 기능이 도입되면, 개발자가 잘못 작성한 Composable은 **치명적인 race condition**이나 **잘못된 UI 결과**로 이어질 수 있다.

#### 병렬 실행이 도입되면 무엇이 문제인가?
Composable 함수가 **서로 다른 스레드에서 동시에 호출될 수 있기 때문**이다. 특히 다음 두 경우가 가장 위험하다.
1. **Composable 내부에서 ViewModel의 상태를 직접 변경하는 경우**
2. **Composable 내부에서 지역 변수(var)를 변경하는 경우

Compose는 Composable을 백그라운드 스레드 풀에서 실행할 수 있다. 이때 같은 Composable이 여러 스레드에서 동시에 호출되면 다음과 같은 현상이 발생한다.
- 증가 연산(`++`)같은 비원자적인 연산이 섞여 race condition 발생
- UI 결과가 의도와 다르게 꼬임
- 내부 상태가 예측 불가능해짐
- Recomposition 타이밍에 따라 값이 계속 달라져 디버깅이 불가능해짐

이것이 Compose가 **\"Composable에 side-effect를 절대 넣지 말라\"**라고 강조하는 이유다.

아래 코드는 공식문서에서 제공한 **Composable 내부에서 로컬 상태를 변경하는 잘못된 예시 코드**이다.
``` kotlin
@Composable
fun ListWithBug(myList: List<String>) {
    var items = 0
    Row(horizontalArrangement = Arrangement.SpaceBetween) {
        Column {
            for (item in myList) {
                Card {
                    Text("Item: $item")
                    items++ // Avoid! Side-effect of the column recomposing.
                }
            }
        }
    }
}
```
이 코드는 두 가지 이유로 잘못되었다.
1. Recomposition 때마다 items가 증가한다
2. 병렬 Recomposition 환경에서는 items 값이 원자적이지 않아 치명적으로 꼬인다

이런 식으로 이론적으로는 10개의 아이템이 있어도 items가 7이 될 수 있고, 20이 될 수도 있다.
그러므로 이러한 위험을 원천 방지하기 위해 **Composable 내부에서 mutable state 변경을 금지하는 것**이다.

## 6. Composable은 미래에 어떤 순서로 실행될지 보장되지 않는다
**현재 Compose는 순서대로 실행되는 것처럼 보인다.** 왜냐하면 현재 Compose는 메인 스레드 단일 실행 모델이기 때문이다.
**하지만 Compose API는 순서를 보장하지 않는다.** 다시 말해서 멀티 스레드가 도입되면 Composable 실행 순서를 API적으로 보장하지 않는다.
















