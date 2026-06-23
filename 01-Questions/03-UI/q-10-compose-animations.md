---
type: question
category: ui
difficulty: senior
tags: [compose, animation]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Compose animations: animate*AsState, Animatable, Transition, AnimatedContent

## Краткий ответ (TL;DR)

`animateFloatAsState` / `animateColorAsState` — простой interpolation между значениями. `Animatable` — низкоуровневое API с явным управлением (suspend `animateTo`, `snapTo`). `Transition` / `updateTransition` — координированные анимации нескольких свойств от одного state. `AnimatedVisibility` / `AnimatedContent` — enter/exit transitions. `rememberInfiniteTransition` — бесконечные.

## Развёрнутый ответ

### animate*AsState

```kotlin
val alpha by animateFloatAsState(
    targetValue = if (visible) 1f else 0f,
    animationSpec = tween(durationMillis = 300)
)
Box(Modifier.graphicsLayer { this.alpha = alpha })
```

Под капотом — `Animatable`. Прост в использовании, но не даёт контроля над прерыванием/cancel.

### Animatable

```kotlin
val offset = remember { Animatable(0f) }

LaunchedEffect(target) {
    offset.animateTo(target, tween(500))
    // suspend: дождётся завершения или cancel при reentrance
}
```

Преимущества:
- `snapTo(value)` — мгновенно (без анимации).
- `stop()` — остановить.
- `velocityCalculator` — fling из текущей скорости.
- Поддерживает любой тип через `TwoWayConverter` (`Animatable(Offset.Zero, Offset.VectorConverter)`).

### updateTransition

```kotlin
val transition = updateTransition(targetState = state, label = "card")

val color by transition.animateColor(label = "color") { s ->
    if (s.selected) Color.Blue else Color.Gray
}
val padding by transition.animateDp(label = "padding") { s ->
    if (s.selected) 16.dp else 8.dp
}
```

Все child-анимации **синхронизированы** (start/end one piece). Хорошо для согласованных переходов.

### AnimatedVisibility

```kotlin
AnimatedVisibility(
    visible = isShown,
    enter = fadeIn() + slideInVertically { -it },
    exit = fadeOut() + slideOutVertically { -it }
) {
    Banner()
}
```

EnterTransition / ExitTransition можно комбинировать через `+`. Контент композирован только когда `visible`.

### AnimatedContent

Crossfade-подобная замена контента с разными transitions для разных направлений:

```kotlin
AnimatedContent(
    targetState = page,
    transitionSpec = {
        if (targetState > initialState) {
            slideInHorizontally { it } togetherWith slideOutHorizontally { -it }
        } else {
            slideInHorizontally { -it } togetherWith slideOutHorizontally { it }
        }
    }
) { p -> Page(p) }
```

### Crossfade

Упрощённая `AnimatedContent`:

```kotlin
Crossfade(targetState = page) { p -> Page(p) }
```

### InfiniteTransition

```kotlin
val transition = rememberInfiniteTransition()
val rotation by transition.animateFloat(
    0f, 360f,
    infiniteRepeatable(tween(2000, easing = LinearEasing))
)
Icon(Modifier.graphicsLayer { rotationZ = rotation })
```

### AnimationSpec

- `tween(duration, delay, easing)` — interpolation.
- `spring(dampingRatio, stiffness)` — физическая пружина (рекомендуется по умолчанию).
- `snap(delayMillis)` — мгновенно.
- `keyframes { 0f at 0; 1f at 100 with LinearOutSlowInEasing; ... }` — по кадрам.
- `repeatable(iterations, animation, repeatMode)` — конечное повторение.

### Производительность

- Читай animated value в **lambda Modifier** (`graphicsLayer { ... }`, `offset { ... }`) → только draw/layout, не recomposition.
- Не оборачивай `animate*AsState` в часто-рекомпозируемый родитель — создаст новые `Animatable` на каждом recomposition. (на самом деле `remember` внутри — но всё равно с key зависящим от target.)
- `Animatable` дешевле `animateFloatAsState` когда нужно несколько последовательных анимаций.

### Shared element transitions

`SharedTransitionLayout` (1.7+) для shared element между screens:

```kotlin
SharedTransitionLayout {
    AnimatedContent(targetState = screen) { s ->
        when (s) {
            is List -> ImageInList(sharedElementOf(this, "img"))
            is Detail -> ImageInDetail(sharedElementOf(this, "img"))
        }
    }
}
```

## Подводные камни

- `animateFloatAsState` без `LaunchedEffect` для запуска чейн → нет контроля над cancel.
- Чтение animation value в composition (не в draw lambda) → recomposes на каждый кадр → жёсткий jank.
- `AnimatedVisibility` с `Modifier.padding` снаружи → padding исчезает не плавно.
- `updateTransition` ожидает stable equality по target state — `data class` ок, `MutableState` нет.
- Использование `InfiniteTransition` для уже-завершившейся анимации → батарея.

## Связанные темы

- [[q-08-compose-modifier]]
- [[q-09-compose-canvas]]
- [[q-19-motion-layout]]

## Follow-up

- Чем `Animatable` отличается от `animateFloatAsState`?
- Когда `updateTransition` лучше, чем несколько `animate*AsState`?
- Почему чтение анимированного значения в lambda Modifier — быстрее?
