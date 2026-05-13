# Layout 속 그라데이션에 대한 고찰

## 개요
숏챠 추천탭 카테고리 칩을 개발하면서 LazyRow 양 옆 ContentPadding 만큼 그라데이션이 들어가는 걸 보고 어느 때처럼 Box에 LazyRow를 넣고 그라데이션을 그릴 Box를 두 개 추가했다.

![image](https://github.com/user-attachments/assets/8b63f09f-ad6d-4657-bf37-e1e0bebf34e6)

```kotlin
Box {
    LazyRow(...)

    // start 그라데이션
    Box(...)

    // end 그라데이션
    Box(...)
}
```

## 아이디어
근데 드는 생각이...

그라데이션을 그릴 때 항상 빈 Box를 사용했는데, 왜 Box를 사용해야하지?   
내부 content도 없이 그냥 정해진 영역에 background만 채우는데, 쓸데 없는 오버헤드가 생기는 거 아닌가?

Layout를 직접 사용해 그라데이션 전용 컴퍼넌트를 만들어 보자.

## Box

근데 Box 정의를 보면 content를 넘기지 않는 Box가 따로 있다.

쓸데 없는 오버헤드도 없는 깔끔한!
```kotlin
@Composable
fun Box(modifier: Modifier) {
    Layout(measurePolicy = EmptyBoxMeasurePolicy, modifier = modifier)
}

internal val EmptyBoxMeasurePolicy = MeasurePolicy { _, constraints ->
    layout(constraints.minWidth, constraints.minHeight) {}
}
```

애초에 우려했던 오버헤드는 없었던 것이다.

## drawWithContent
하지만 양 옆에 그라데이션을 추가하기 위해 여전히 Box가 필요하고 가독성은 떨진다.   
또 recomposition이 일어날 때 매번 무거운 Brush.verticalGradient 를 실행하고 있다. (물론 remember로 감싸면 되지만)


그래서 고민 결과 LazyRow에 drawWithContent를 사용했다.

LazyRow를 Box로 감쌀 필요도 없고 drawWithContent는 compose 단계 중에서도 Drawing 단계에서 Compose.State를 관찰하기 때문에 불필요한 recomposition를 건너띌 수 있다.

<img width="844" alt="image" src="https://github.com/user-attachments/assets/06fe2d97-9a11-48b0-a7ad-b60aaec4e3d9" />


## 결과

```kotlin
LazyRow(
    modifier = modifier
        .fillMaxSize()
        .horizontalFadingEdges(padding = horizontalPadding)
)
```

```kotlin
fun Modifier.horizontalFadingEdges(
    padding: Dp
): Modifier = this.drawWithContent {
    drawContent()
    
    drawEndGradient(padding)

    drawStartGradient(padding)
}
```
