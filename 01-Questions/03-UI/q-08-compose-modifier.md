---
type: question
category: ui
difficulty: senior
tags: [compose, modifier, modifier-node]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Compose Modifier: chain, composed vs Modifier.Node, кастомные модификаторы

## Краткий ответ (TL;DR)

`Modifier` — иммутабельная цепочка декораторов layout-узла. Применяется слева направо, **порядок имеет смысл** (padding, size, background и т.д.). Старый `Modifier.composed { ... }` создавал новый экземпляр на каждой recomposition (бьёт по skipping). Современный подход — `Modifier.Node` API (1.6+): аллоцируется один раз, перезаписывает параметры через `update()`.

## Развёрнутый ответ

### Чем Modifier является

```kotlin
@Stable interface Modifier {
    fun <R> foldIn(initial: R, op: (R, Modifier.Element) -> R): R
    // ...
}
```

Цепочка элементов; рантайм при layout/draw проходит по списку. `Modifier.then(other)` — конкатенация.

### Порядок

```kotlin
Modifier
    .clickable { ... }       // 1. ловит клик
    .padding(8.dp)           // 2. отступ от внешнего
    .background(Red)         // 3. фон ПОД остальным
    .padding(8.dp)           // 4. внутренний padding
    .size(100.dp)            // 5. фиксированный размер контента
```

`background` ставится в позиции цепочки → красный закроет область, которая «снаружи» от него. Менять порядок — менять визуал.

### Modifier.composed (старый API)

```kotlin
fun Modifier.shimmer(): Modifier = composed {
    val alpha by animateFloatAsState(...)
    this.graphicsLayer { this.alpha = alpha }
}
```

Создаёт **новую цепочку** на каждой recomposition родителя → ломает skipping (параметры composable становятся unstable).

### Modifier.Node (новый API, 1.6+)

```kotlin
class ShimmerNode : Modifier.Node(), DrawModifierNode {
    var color: Color = Color.Gray
    override fun ContentDrawScope.draw() {
        drawContent()
        drawRect(color, alpha = 0.3f)
    }
}

class ShimmerElement(val color: Color) : ModifierNodeElement<ShimmerNode>() {
    override fun create() = ShimmerNode().also { it.color = color }
    override fun update(node: ShimmerNode) { node.color = color }
    override fun hashCode() = color.hashCode()
    override fun equals(other: Any?) = (other as? ShimmerElement)?.color == color
}

fun Modifier.shimmer(color: Color = Color.Gray) = this then ShimmerElement(color)
```

Преимущества:
- **Stable** — `equals/hashCode` корректные → composable остаётся skippable.
- Node живёт между recomposition'ами; нет аллокаций каждый раз.
- Можно наследовать `DrawModifierNode`, `LayoutModifierNode`, `PointerInputModifierNode`, `ObserverModifierNode` и др.

### Полезные built-in modifiers

| Modifier | Что делает |
|----------|-----------|
| `padding(8.dp)` | внешний/внутренний отступ |
| `size(w, h)` / `width()` / `height()` | задаёт размер |
| `fillMaxWidth()` / `fillMaxSize()` | растягивает по родителю |
| `wrapContentSize()` | сворачивает до контента |
| `background(color)` / `border()` | фон, рамка |
| `clip(shape)` | обрезка по форме |
| `graphicsLayer { ... }` | альфа, ротация, трансляция (на отдельном layer'е) |
| `clickable`, `combinedClickable`, `pointerInput` | input |
| `focusable`, `onFocusChanged` | фокус |
| `semantics { ... }` | accessibility / тест |
| `testTag("...")` | для UI-тестов |

### graphicsLayer

Включает hardware-accelerated layer → дешёвые анимации (alpha, rotation, translation). Использовать вместо изменения color/offset в composition layer.

```kotlin
Modifier.graphicsLayer {
    alpha = animatedAlpha
    rotationZ = angle
    translationX = scrollOffset
}
```

Lambda → читается в **draw**-фазе, не в composition → recomposition не триггерится.

### pointerInput

```kotlin
Modifier.pointerInput(Unit) {
    detectDragGestures { _, drag -> offset += drag }
}
```

Key (`Unit`/`obj`) — при изменении пересоздаётся coroutine с gesture detector.

### Modifier.then vs composed

`then` — статическая композиция, дешёвая.
`composed` — динамическая (содержит @Composable), для legacy.

## Подводные камни

- `composed { ... }` → нестабильный modifier параметр → ломает skipping. Migrate на Node.
- Порядок modifier: `size().padding()` vs `padding().size()` — разный результат.
- `pointerInput` без key (или с нестабильным) — перезапуск detector на каждой recomposition.
- `graphicsLayer { alpha = state.value }` — чтение через lambda → ок. `graphicsLayer(alpha = state.value)` (overload без lambda) — чтение в composition → recomposes.
- `Modifier.clip(shape)` без `graphicsLayer` — может быть дорогой; следить за clipping в perf.

## Связанные темы

- [[q-05-compose-performance]]
- [[q-07-compose-layouts]]
- [[q-09-compose-canvas]]

## Follow-up

- Чем `Modifier.Node` лучше `Modifier.composed`?
- Почему `Modifier.graphicsLayer { alpha = a }` дешевле, чем `Modifier.alpha(a)`?
- Зачем `key` в `pointerInput`?
