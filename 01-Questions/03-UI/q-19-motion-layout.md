---
type: question
category: ui
difficulty: middle
tags: [animation, property-animation, view-system]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Анимации в View-системе: ObjectAnimator, ValueAnimator, Transition API

## Краткий ответ (TL;DR)

В View-системе: **`ObjectAnimator`** анимирует property (`alpha`, `translationX`) на view; **`ValueAnimator`** даёт interpolated value, который можно применить ручно; **`AnimatorSet`** — оркестрация. Для переходов между layout — **`TransitionManager.beginDelayedTransition()`**, **`Scene`**, **`MotionLayout`** (см. [[q-15-constraintlayout]]). Frame-by-frame через `Animation`/`AnimationDrawable` — устаревший pre-Honeycomb API.

## Развёрнутый ответ

### ObjectAnimator

```kotlin
ObjectAnimator.ofFloat(view, "translationX", 0f, 300f).apply {
    duration = 500
    interpolator = AccelerateDecelerateInterpolator()
    start()
}

// Несколько свойств
ObjectAnimator.ofPropertyValuesHolder(
    view,
    PropertyValuesHolder.ofFloat("alpha", 0f, 1f),
    PropertyValuesHolder.ofFloat("scaleX", 0.5f, 1f)
).start()
```

### ValueAnimator

```kotlin
ValueAnimator.ofInt(0, 100).apply {
    duration = 1000
    addUpdateListener { animator ->
        progressBar.progress = animator.animatedValue as Int
    }
    start()
}
```

### AnimatorSet

```kotlin
AnimatorSet().apply {
    playSequentially(fadeIn, scaleUp)  // или playTogether(...)
    start()
}
```

### ViewPropertyAnimator (fluent API)

```kotlin
view.animate()
    .alpha(0f)
    .translationX(100f)
    .setDuration(300)
    .withEndAction { view.isGone = true }
    .start()
```

### Spring animation (Physics)

```kotlin
SpringAnimation(view, SpringAnimation.TRANSLATION_X).apply {
    spring = SpringForce(0f)
        .setDampingRatio(SpringForce.DAMPING_RATIO_MEDIUM_BOUNCY)
        .setStiffness(SpringForce.STIFFNESS_LOW)
    setStartValue(300f)
    start()
}
```

`androidx.dynamicanimation` — физические анимации (spring, fling). Естественные движения.

### Transition API

```kotlin
TransitionManager.beginDelayedTransition(rootViewGroup, AutoTransition())
view.isGone = true   // ChangeBounds + Fade анимируется автоматически
```

`AutoTransition` = ChangeBounds + Fade + ChangeTransform. Можно собрать вручную:

```kotlin
val transition = TransitionSet().apply {
    addTransition(ChangeBounds())
    addTransition(Fade())
    duration = 300
}
TransitionManager.beginDelayedTransition(root, transition)
```

### Shared element transition (Activity)

```kotlin
val options = ActivityOptions.makeSceneTransitionAnimation(
    activity,
    Pair(imageView, "image"),
    Pair(titleView, "title")
)
startActivity(intent, options.toBundle())
```

В принимающей Activity: те же `transitionName`.

### Interpolators

- `AccelerateInterpolator`, `DecelerateInterpolator`, `AccelerateDecelerateInterpolator`.
- `OvershootInterpolator`, `BounceInterpolator`, `AnticipateInterpolator`.
- `LinearInterpolator`.
- `PathInterpolator` — кастомная кривая Bezier.

### Performance

- Property animations работают на UI thread, но используют `RenderThread` для apply.
- `translationX/Y`, `alpha`, `rotation`, `scaleX/Y` — hardware-accelerated через `RenderNode`/displayList.
- `layout` параметры (width/height/margin) → `requestLayout` → дорого. Анимируй через `scaleX` если можно.

### MotionLayout vs Property Animation

| Случай | Что использовать |
|--------|------------------|
| Одно свойство, fire-and-forget | ObjectAnimator / ViewPropertyAnimator |
| Координированные переходы между layout state | MotionLayout |
| Анимация после `setVisibility` | TransitionManager.beginDelayedTransition |
| Natural motion / springs | SpringAnimation |

В Compose всё через Animatable/Transition (см. [[q-10-compose-animations]]).

## Подводные камни

- `ObjectAnimator.ofFloat(view, "width", ...)` — работает (через reflection), но дорого: вызовет `requestLayout`.
- Animator running после `onDestroy` view → лик. `view.animate().cancel()` в onPause.
- `Fade` transition без `setVisibility` change → не сработает.
- `beginDelayedTransition` для `RecyclerView` items — конфликтует с `ItemAnimator`.
- Shared element transition не работает с `windowAnimationStyle = none`.

## Связанные темы

- [[q-10-compose-animations]]
- [[q-15-constraintlayout]]
- [[q-13-views-measure-layout-draw]]

## Follow-up

- Чем `ObjectAnimator` отличается от `ValueAnimator`?
- Зачем `TransitionManager.beginDelayedTransition`?
- Почему анимировать `width`/`height` плохо?
