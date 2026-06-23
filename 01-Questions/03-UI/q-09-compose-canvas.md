---
type: question
category: ui
difficulty: middle
tags: [compose, canvas, drawing]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Compose Canvas, drawBehind, drawWithCache, графика

## Краткий ответ (TL;DR)

`Canvas { ... }` — composable для произвольного рисования. `Modifier.drawBehind { ... }` — рисует под контентом без отдельного элемента. `Modifier.drawWithCache { onDrawBehind { ... } }` — кэширует expensive объекты (Path, Brush) между перерисовками. Все три используют `DrawScope` (size, drawRect/drawCircle/drawPath/drawImage).

## Развёрнутый ответ

### Canvas composable

```kotlin
Canvas(modifier = Modifier.size(120.dp)) {
    val r = size.minDimension / 2f
    drawCircle(color = Color.Red, radius = r, center = center)
}
```

`DrawScope` даёт: `size`, `center`, методы `drawRect/Circle/Line/Path/Image/Text/Arc`.

### drawBehind / drawWithContent

```kotlin
Box(
    Modifier
        .size(100.dp)
        .drawBehind {
            drawRect(Color.Yellow)   // под содержимым Box
        }
) { Text("Hi") }

Modifier.drawWithContent {
    drawContent()                     // оригинальный контент
    drawRect(Color.Red.copy(alpha = 0.3f))   // overlay
}
```

### drawWithCache

Объекты типа `Path`, `LinearGradient`, `ImageBitmap` дорого создавать. `drawWithCache` пересоздаёт их **только при изменении размера** (или указанных keys).

```kotlin
Modifier.drawWithCache {
    val brush = Brush.linearGradient(listOf(Color.Red, Color.Blue))
    val path = Path().apply {
        moveTo(0f, 0f); lineTo(size.width, size.height)
    }
    onDrawBehind {
        drawPath(path, brush)
    }
}
```

### graphicsLayer для трансформаций

Не делать transforms в Canvas через `withTransform` если можно через `Modifier.graphicsLayer` — graphicsLayer hardware-accelerated.

### Текст в Canvas

```kotlin
val textMeasurer = rememberTextMeasurer()
Canvas(Modifier.fillMaxSize()) {
    val layout = textMeasurer.measure(AnnotatedString("Hello"))
    drawText(layout, topLeft = Offset(10f, 10f))
}
```

### Path

```kotlin
val path = Path().apply {
    moveTo(0f, 0f)
    quadraticBezierTo(50f, -50f, 100f, 0f)
    close()
}
drawPath(path, Color.Blue, style = Stroke(width = 4f))
```

`Stroke(width, cap, join, pathEffect)` — обводка; без `style` — fill.

### Animation в Canvas

```kotlin
val progress by rememberInfiniteTransition().animateFloat(
    0f, 1f, infiniteRepeatable(tween(1000))
)
Canvas(Modifier.size(80.dp)) {
    drawArc(Color.Red, startAngle = 0f, sweepAngle = 360f * progress,
            useCenter = false, style = Stroke(8f))
}
```

`progress` читается в draw-lambda → нет recomposition, только redraw.

### Compose vs Android Canvas

`androidx.compose.ui.graphics.Canvas` — обёртка над `android.graphics.Canvas`. Через `drawIntoCanvas { canvas -> canvas.nativeCanvas.drawX(...) }` можно использовать нативные методы (если в Compose нет аналога).

## Подводные камни

- Создание `Path`/`Brush` внутри `onDraw` без cache → аллокация каждую кадр → GC pressure.
- Использование `LocalDensity` для конвертации dp→px: в DrawScope уже доступен `Density` через `this`.
- Большой Path сложной формы → дорогой draw; рассмотри `Bitmap` cache.
- `Canvas` без явного `Modifier.size(...)` → 0×0 (нет размера) — не нарисуется.
- `drawWithCache` пересоздаёт block при изменении size — если size меняется анимированно, кэш не помогает.

## Связанные темы

- [[q-07-compose-layouts]]
- [[q-08-compose-modifier]]
- [[q-10-compose-animations]]

## Follow-up

- Чем `drawBehind` отличается от `Canvas`?
- Зачем `drawWithCache` если есть `drawBehind`?
- Почему нельзя создавать `Path` внутри `onDraw`?
