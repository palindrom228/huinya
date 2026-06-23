---
type: question
category: ui
difficulty: senior
tags: [view-system, measure, layout, draw]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# View system: measure / layout / draw, MeasureSpec, requestLayout vs invalidate

## Краткий ответ (TL;DR)

View имеет 3 фазы: **measure** (вычислить размер, ViewGroup передаёт `MeasureSpec` детям), **layout** (расставить координаты `left/top/right/bottom`), **draw** (вызвать `onDraw(Canvas)`). `requestLayout()` запускает все три (дорогая операция, поднимается до root); `invalidate()` — только draw. RecyclerView/ConstraintLayout — оптимизированные ViewGroup, минимизирующие перерасчёты.

## Развёрнутый ответ

### MeasureSpec

```kotlin
val mode = MeasureSpec.getMode(spec)   // EXACTLY, AT_MOST, UNSPECIFIED
val size = MeasureSpec.getSize(spec)
```

| Mode | Что значит |
|------|-----------|
| EXACTLY | размер точно `size` (match_parent, конкретные dp) |
| AT_MOST | не больше `size` (wrap_content) |
| UNSPECIFIED | без ограничений (ScrollView, RecyclerView к детям) |

ViewGroup в `onMeasure` дёргает `measureChild(child, parentSpec)` → child вычисляет себя и вызывает `setMeasuredDimension(w, h)`.

```kotlin
override fun onMeasure(widthSpec: Int, heightSpec: Int) {
    var w = 0; var h = 0
    for (i in 0 until childCount) {
        val c = getChildAt(i)
        measureChild(c, widthSpec, heightSpec)
        w = maxOf(w, c.measuredWidth)
        h += c.measuredHeight
    }
    setMeasuredDimension(
        resolveSize(w, widthSpec),
        resolveSize(h, heightSpec)
    )
}
```

### Layout

```kotlin
override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
    var y = 0
    for (i in 0 until childCount) {
        val c = getChildAt(i)
        c.layout(0, y, c.measuredWidth, y + c.measuredHeight)
        y += c.measuredHeight
    }
}
```

### Draw pipeline

```
draw(canvas)
  → drawBackground
  → onDraw(canvas)              ← наш контент
  → dispatchDraw(canvas)        ← рисует child views (для ViewGroup)
  → onDrawForeground(canvas)
```

`onDraw` для **View**, `dispatchDraw` для **ViewGroup**.

### requestLayout vs invalidate

- `invalidate()` — пометить как «нужен redraw», следующий vsync — draw фаза.
- `requestLayout()` — нужен новый measure + layout + draw. Поднимается до root → может быть очень дорого.

Если меняешь только цвет/прозрачность — `invalidate()`. Если изменился размер контента → `requestLayout()`.

### Двойной measure pass

В View-системе ViewGroup может вызвать `measure` ребёнка несколько раз (например, RelativeLayout, LinearLayout с weight). Это причина jank'а сложных layout'ов. ConstraintLayout пытается это сделать в один проход.

Compose — **single-pass measure** by design.

### MotionEvent dispatch

```
dispatchTouchEvent → onInterceptTouchEvent (ViewGroup) → onTouchEvent (child) → onTouchEvent (parent)
```

ViewGroup может перехватить через `onInterceptTouchEvent` (return true → дальше child не получает).

### setWillNotDraw

ViewGroup по умолчанию не вызывает `onDraw` — оптимизация. Если рисуешь — `setWillNotDraw(false)` в конструкторе.

### LayoutInflater

XML → View tree через reflection (медленно). `AsyncLayoutInflater` или `LayoutInflater.Factory2` ускоряют. В Compose проблемы нет.

### Hardware acceleration

С API 14 по умолчанию включено. View рендерится через `DisplayList` (записан в GPU команды). Некоторые ops (Canvas.drawPath с PathEffect) не аппаратно ускоряются — fallback на CPU layer.

### View invalidation regions

`invalidate(rect)` — invalidate только этот регион (оптимизация). Полезно в кастомных View с большой площадью.

## Подводные камни

- `requestLayout()` в `onLayout` → бесконечный цикл (или silent skip + warning).
- `View.GONE` в LinearLayout с `weight` → перерасчёт всего парента.
- `WRAP_CONTENT` в RecyclerView item-root → bad perf (multiple measure).
- Тяжёлый `onDraw` (Path, аллокации) → jank, GC.
- `View.post { requestLayout() }` — частый хак, обычно говорит о неправильном лайфтайме.

## Связанные темы

- [[q-14-recyclerview]]
- [[q-15-constraintlayout]]
- [[../07-Performance-Memory/q-jank]]

## Follow-up

- Чем `requestLayout` отличается от `invalidate`?
- Почему `LinearLayout` с weight может вызывать двойной measure?
- Что такое `MeasureSpec` и его три mode'а?
