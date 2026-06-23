---
type: question
category: ui
difficulty: senior
tags: [compose, interop, view-system]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Compose ↔ View interop: ComposeView, AndroidView, AbstractComposeView, ViewCompositionStrategy

## Краткий ответ (TL;DR)

`ComposeView` встраивает Compose в XML/Fragment; `AndroidView { factory }` — View в Compose. Lifecycle composition контролируется через `ViewCompositionStrategy` (default `DisposeOnDetachedFromWindow` — но во **Fragment с `viewLifecycleOwner`** нужен `DisposeOnViewTreeLifecycleDestroyed`). `AbstractComposeView` — база для своих переиспользуемых composable-view'ев.

## Развёрнутый ответ

### Compose в XML/Fragment

```xml
<androidx.compose.ui.platform.ComposeView
    android:id="@+id/compose_view"
    android:layout_width="match_parent"
    android:layout_height="wrap_content" />
```

```kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    val composeView = view.findViewById<ComposeView>(R.id.compose_view)
    composeView.setViewCompositionStrategy(
        ViewCompositionStrategy.DisposeOnViewLifecycleDestroyed(viewLifecycleOwner)
    )
    composeView.setContent {
        AppTheme { MyScreen() }
    }
}
```

`DisposeOnViewLifecycleDestroyed(viewLifecycleOwner)` — освобождает composition когда Fragment view destroyed, что важно для Fragment-в-ViewPager (View destroyed, Fragment жив).

### ViewCompositionStrategy опции

- `DisposeOnDetachedFromWindow` — default; работает для большинства Activity.
- `DisposeOnDetachedFromWindowOrReleasedFromPool` — для items в RecyclerView/ViewPager2.
- `DisposeOnLifecycleDestroyed(lifecycle)`.
- `DisposeOnViewTreeLifecycleDestroyed` — auto-detect через `ViewTreeLifecycleOwner` (рекомендуется чаще всего).

### AndroidView (View в Compose)

```kotlin
AndroidView(
    factory = { ctx ->
        WebView(ctx).apply {
            settings.javaScriptEnabled = true
        }
    },
    update = { webView ->
        webView.loadUrl(url)   // вызывается при изменении state
    },
    onRelease = { webView ->
        webView.destroy()
    }
)
```

`factory` — создаётся один раз; `update` — на каждом recomposition; `onRelease` — при удалении composable.

### AndroidViewBinding (для XML layouts)

```kotlin
AndroidViewBinding(MyXmlLayoutBinding::inflate) {
    titleTextView.text = "Hello"
}
```

Удобно для inflating XML внутри Compose.

### AbstractComposeView

```kotlin
class GreetingView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : AbstractComposeView(context, attrs, defStyleAttr) {

    var name by mutableStateOf("World")

    @Composable
    override fun Content() {
        Text("Hello $name")
    }
}
```

Кастомный View, который рендерится через Compose. Можно вставлять в XML как обычный View.

### CompositionLocal propagation

`ComposeView` автоматически использует `LocalContext`, `LocalLifecycleOwner` (через `ViewTreeLifecycleOwner`), `LocalSavedStateRegistryOwner`, `LocalView`. Установлены через `ViewTree*Owner` setter'ы.

В Fragment-based навигации: `viewLifecycleOwner` пробрасывается автоматически через `ViewTreeLifecycleOwner.set(view, viewLifecycleOwner)`.

### Fragment-Compose pattern

```kotlin
class MyFragment : Fragment() {
    override fun onCreateView(...): View {
        return ComposeView(requireContext()).apply {
            setViewCompositionStrategy(
                ViewCompositionStrategy.DisposeOnViewLifecycleDestroyed(viewLifecycleOwner)
            )
            setContent {
                MyScreen(viewModel = hiltViewModel())
            }
        }
    }
}
```

Без явного `setViewCompositionStrategy` Fragment в `ViewPager2` будет течь: composition не освобождается при `onDestroyView`.

### Compose в RecyclerView item

```kotlin
class MyViewHolder(val composeView: ComposeView) : RecyclerView.ViewHolder(composeView) {
    init {
        composeView.setViewCompositionStrategy(
            ViewCompositionStrategy.DisposeOnDetachedFromWindowOrReleasedFromPool
        )
    }
    fun bind(item: Item) {
        composeView.setContent { ItemRow(item) }
    }
}
```

Дорого! Перфоманс хуже, чем native Compose `LazyColumn`. Использовать только при миграции.

### Theme bridging

Если у вас есть XML тема и хотите её цвета в Compose:

```kotlin
MdcTheme {   // com.google.android.material:compose-theme-adapter (legacy)
    MyComposable()
}
```

Современный подход — определи `MaterialTheme` явно в Compose, не bridge'и.

## Подводные камни

- `ComposeView` в Fragment + ViewPager2 без правильного strategy → memory leak через composition.
- `AndroidView` с тяжёлым `factory` каждый recomposition — нет, `factory` вызывается раз, но `update` каждый. Тяжёлые операции вынести в effect.
- `AbstractComposeView` нельзя inflate из XML до того, как `ViewTreeLifecycleOwner` установлен. В кастомной ситуации — `setViewTreeLifecycleOwner` вручную.
- Compose-в-RecyclerView item: composition создаётся для каждого item, не recycle Composer — заметный overhead.
- Theme: `LocalContext.current` достаёт `Context`, но `MaterialTheme.colorScheme` — это Compose theme, не XML. Не смешивать.

## Связанные темы

- [[q-02-compose-state]]
- [[q-13-views-measure-layout-draw]]
- [[../02-Android-Core/q-02-fragment-lifecycle]]

## Follow-up

- Почему `DisposeOnDetachedFromWindow` не подходит для Fragment во ViewPager2?
- Что делает `AndroidView { factory, update }`?
- Когда `AbstractComposeView` лучше, чем `ComposeView.setContent`?
