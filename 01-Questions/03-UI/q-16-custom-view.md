---
type: question
category: ui
difficulty: senior
tags: [custom-view, view-system]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Кастомные View: extend, custom attrs, save state, accessibility

## Краткий ответ (TL;DR)

Три пути: (1) **compound view** (наследуем ViewGroup, inflate layout), (2) **custom drawing** (extend View, реализуем `onDraw`), (3) **custom ViewGroup** (extend ViewGroup, реализуем `onMeasure`+`onLayout`). Custom attrs объявляются в `res/values/attrs.xml`. Сохранение state — `onSaveInstanceState`/`onRestoreInstanceState` с `BaseSavedState`. Accessibility — `setAccessibilityDelegate`.

## Развёрнутый ответ

### Custom View (drawing)

```kotlin
class CircleView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {

    private val paint = Paint(Paint.ANTI_ALIAS_FLAG)
    private var circleColor: Int = Color.RED
        set(value) { field = value; paint.color = value; invalidate() }

    init {
        context.withStyledAttributes(attrs, R.styleable.CircleView) {
            circleColor = getColor(R.styleable.CircleView_circleColor, Color.RED)
        }
        paint.color = circleColor
    }

    override fun onDraw(canvas: Canvas) {
        val r = minOf(width, height) / 2f
        canvas.drawCircle(width / 2f, height / 2f, r, paint)
    }
}
```

### Custom attrs

`res/values/attrs.xml`:

```xml
<resources>
    <declare-styleable name="CircleView">
        <attr name="circleColor" format="color" />
        <attr name="circleRadius" format="dimension" />
        <attr name="circleStyle">
            <enum name="fill" value="0" />
            <enum name="stroke" value="1" />
        </attr>
    </declare-styleable>
</resources>
```

Использование:

```xml
<com.x.CircleView
    app:circleColor="@color/red"
    app:circleStyle="stroke" />
```

### onMeasure для custom View

Если стандартный `View.onMeasure` (берёт MeasureSpec.size при EXACTLY/AT_MOST, иначе minWidth) не подходит:

```kotlin
override fun onMeasure(widthSpec: Int, heightSpec: Int) {
    val desiredWidth = paddingLeft + paddingRight + ...
    val desiredHeight = paddingTop + paddingBottom + ...
    setMeasuredDimension(
        resolveSize(desiredWidth, widthSpec),
        resolveSize(desiredHeight, heightSpec)
    )
}
```

### onSaveInstanceState

```kotlin
override fun onSaveInstanceState(): Parcelable {
    val superState = super.onSaveInstanceState()
    return SavedState(superState).also { it.value = currentValue }
}

override fun onRestoreInstanceState(state: Parcelable?) {
    val ss = state as? SavedState
    super.onRestoreInstanceState(ss?.superState ?: state)
    ss?.let { currentValue = it.value }
}

@Parcelize
class SavedState(
    val superState: Parcelable?,
    var value: Int = 0
) : View.BaseSavedState(superState)
```

View **обязана иметь `android:id`** для сохранения state — иначе игнорируется.

### Custom ViewGroup

```kotlin
class FlowLayout @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null
) : ViewGroup(context, attrs) {

    override fun onMeasure(wSpec: Int, hSpec: Int) {
        val width = MeasureSpec.getSize(wSpec)
        var rowWidth = 0; var rowHeight = 0; var totalHeight = 0
        for (i in 0 until childCount) {
            val c = getChildAt(i)
            measureChild(c, wSpec, hSpec)
            if (rowWidth + c.measuredWidth > width) {
                totalHeight += rowHeight; rowWidth = 0; rowHeight = 0
            }
            rowWidth += c.measuredWidth
            rowHeight = maxOf(rowHeight, c.measuredHeight)
        }
        totalHeight += rowHeight
        setMeasuredDimension(width, totalHeight)
    }

    override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) { ... }
}
```

### Touch handling

```kotlin
override fun onTouchEvent(e: MotionEvent): Boolean {
    when (e.action) {
        MotionEvent.ACTION_DOWN -> { ...; return true }   // return true to receive next events
        MotionEvent.ACTION_MOVE -> ...
        MotionEvent.ACTION_UP -> { performClick(); return true }
    }
    return super.onTouchEvent(e)
}
```

`performClick()` — обязательно для accessibility.

### Accessibility

```kotlin
ViewCompat.setAccessibilityDelegate(view, object : AccessibilityDelegateCompat() {
    override fun onInitializeAccessibilityNodeInfo(host: View, info: AccessibilityNodeInfoCompat) {
        super.onInitializeAccessibilityNodeInfo(host, info)
        info.roleDescription = "slider"
        info.contentDescription = "Volume $value"
        info.addAction(AccessibilityActionCompat.ACTION_CLICK)
    }
})
```

### Compose interop

```kotlin
// Compose в XML
<androidx.compose.ui.platform.ComposeView android:id="@+id/compose" />

view.findViewById<ComposeView>(R.id.compose).setContent {
    MyComposable()
}

// XML View в Compose
AndroidView(factory = { ctx -> MyCustomView(ctx) }, update = { it.value = x })
```

## Подводные камни

- Без `@JvmOverloads` (или ручных конструкторов) — `View(context, attrs)` не подцепится при inflate.
- `onSaveInstanceState` без `id` у View → state теряется.
- `onDraw` с аллокациями (new Paint каждый раз) → GC.
- `onTouchEvent` без `performClick()` — Lint warning + сломан a11y.
- Hardware-acceleration: некоторые `Paint`/`Canvas` методы не поддерживаются → нужен `setLayerType(LAYER_TYPE_SOFTWARE, null)`.

## Связанные темы

- [[q-13-views-measure-layout-draw]]
- [[q-09-compose-canvas]]
- [[q-17-accessibility]]

## Follow-up

- Как сохранить state кастомной View через rotation?
- Зачем `performClick()` в `onTouchEvent`?
- Какие три «вида» кастомных View?
