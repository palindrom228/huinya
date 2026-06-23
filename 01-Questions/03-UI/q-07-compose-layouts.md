---
type: question
category: ui
difficulty: senior
tags: [compose, layout, modifier]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Compose layouts: Row/Column/Box, custom Layout, SubcomposeLayout, IntrinsicMeasurements

## Краткий ответ (TL;DR)

Базовые: `Row`/`Column` (linear), `Box` (overlay). Constraints text-приоритетные (без weights). Свой layout — `Layout(content) { measurables, constraints -> layout(w, h) { placeables.place(...) } }`. `SubcomposeLayout` — когда дети **зависят от размера других детей** (например, IntrinsicSize Tab). Тяжелее обычного — не злоупотреблять.

## Развёрнутый ответ

### Single-pass measurement

Compose-layout — **один проход** measure (в отличие от View, где `measure` могут вызвать дважды). Каждый child measured **один раз**. Это значит:

- Нет двойного прогона как `wrap_content` ↔ `match_parent` смешения.
- Нельзя «спросить размер ребёнка, потом изменить constraints» в обычном layout — для этого SubcomposeLayout.

### Constraints

```kotlin
data class Constraints(val minWidth: Int, val maxWidth: Int,
                       val minHeight: Int, val maxHeight: Int)
```

Передаются вниз. Modifier `fillMaxWidth()` ставит `minWidth = maxWidth`. `wrapContentSize()` снимает min.

### Modifier order — это важно

Modifier применяется **слева направо** в композиции layout-цепочки, но **снаружи внутрь** в плане размеров (родительский modifier видит ребёнка обёрнутого).

```kotlin
Modifier
    .size(100.dp)            // обещает 100×100
    .padding(8.dp)           // от 100 отдаст 84 контенту
    .background(Red)         // фон под padding (т.е. 100×100 красный)

Modifier
    .padding(8.dp)
    .size(100.dp)            // 100×100 контент, плюс padding снаружи → итого 116×116
    .background(Red)         // фон 100×100 (без padding)
```

### Weight (только в Row/Column)

```kotlin
Row {
    Box(Modifier.weight(1f))
    Box(Modifier.weight(2f))
}
```

Внутри Row/Column children сначала меряются без weight, потом оставшееся пространство делится по weight'ам.

### Custom Layout

```kotlin
@Composable
fun VStack(modifier: Modifier = Modifier, content: @Composable () -> Unit) {
    Layout(content, modifier) { measurables, constraints ->
        val placeables = measurables.map { it.measure(constraints) }
        val width = placeables.maxOf { it.width }
        val height = placeables.sumOf { it.height }
        layout(width, height) {
            var y = 0
            placeables.forEach { it.place(0, y); y += it.height }
        }
    }
}
```

Чистый и дешёвый — один проход measure + один place.

### SubcomposeLayout

Когда нужно сначала измерить часть детей, потом subcompose другую:

```kotlin
SubcomposeLayout { constraints ->
    val mainPlaceables = subcompose("main", mainContent).map { it.measure(constraints) }
    val mainHeight = mainPlaceables.maxOf { it.height }

    val footerPlaceables = subcompose("footer") {
        Footer(height = mainHeight)   // зависит от main
    }.map { it.measure(constraints) }

    layout(...) { ... }
}
```

Используется в `BoxWithConstraints`, `LazyColumn`, `Scaffold` (где topbar/bottombar измерены перед content). **Дороже** обычного Layout — другой compose subtree.

### BoxWithConstraints

```kotlin
BoxWithConstraints {
    if (maxWidth < 600.dp) Phone() else Tablet()
}
```

Реализован через SubcomposeLayout. Использовать когда **реально нужен размер для выбора composable**, а не просто modifier.

### Intrinsic measurements

```kotlin
Row(Modifier.height(IntrinsicSize.Min)) {
    Text("Long text...")
    Divider(Modifier.fillMaxHeight().width(1.dp))
    Text("Short")
}
```

Спрашиваем min/max intrinsic height у Row перед фактическим measure → Divider растягивается на высоту максимального текста. Children должны поддерживать intrinsic (большинство built-ins — да).

### Alignment / Arrangement

```kotlin
Row(
    horizontalArrangement = Arrangement.SpaceBetween,
    verticalAlignment = Alignment.CenterVertically
) { ... }
```

`Arrangement.spacedBy(8.dp)` — гэп между детьми (популярная замена ручному padding).

### ConstraintLayout

```kotlin
ConstraintLayout {
    val (a, b) = createRefs()
    Box(Modifier.constrainAs(a) { top.linkTo(parent.top) })
    Box(Modifier.constrainAs(b) { top.linkTo(a.bottom); start.linkTo(parent.start) })
}
```

В Compose редко нужен — обычно `Row`/`Column`/`Box` хватает. Использовать когда **сложные взаимные связи** или анимация через `MotionLayout`.

## Подводные камни

- `BoxWithConstraints` дорог — не оборачивать им весь экран без нужды.
- `IntrinsicSize.Min/Max` требует children, которые умеют intrinsic → custom layout должен реализовать `minIntrinsicHeight` и др.
- `Modifier.size().wrapContentSize()` — порядок важен (последний выигрывает).
- `weight` работает только внутри Row/Column, в Box — нет.
- SubcomposeLayout рекомпозирует subtree на каждый measure — не для часто меняющихся subtree.

## Связанные темы

- [[q-08-compose-modifier]]
- [[q-09-compose-canvas]]
- [[q-13-views-measure-layout-draw]]

## Follow-up

- Чем `Layout` отличается от `SubcomposeLayout`?
- Почему Compose single-pass measurement, а View — нет?
- Что делает `Modifier.height(IntrinsicSize.Min)`?
