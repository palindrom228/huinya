---
type: question
category: ui
difficulty: middle
tags: [compose, state, remember]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Compose state: remember, mutableStateOf, derivedStateOf, rememberSaveable

## Краткий ответ (TL;DR)

`mutableStateOf` — наблюдаемый контейнер значения. `remember` — кэширует значение между рекомпозициями (по позиции). `rememberSaveable` — переживает configuration change и process death (через `Bundle`/`Saver`). `derivedStateOf` — оптимизация: рекомпозирует только если вычисленный результат реально изменился.

## Развёрнутый ответ

### remember vs rememberSaveable

```kotlin
val a by remember { mutableStateOf("") }            // потеряется при rotation
val b by rememberSaveable { mutableStateOf("") }    // переживёт rotation + process death
```

`rememberSaveable` сериализует в `Bundle` через автоматический `Saver`. Для кастомных типов:

```kotlin
val state = rememberSaveable(saver = MyStateSaver) { MyState() }

val MyStateSaver = Saver<MyState, Bundle>(
    save = { Bundle().apply { putString("x", it.x) } },
    restore = { MyState(it.getString("x").orEmpty()) }
)
```

Или для `data class` — `listSaver` / `mapSaver`.

### key-based remember

```kotlin
val parsed = remember(input) { parse(input) }
```

При изменении `input` блок пересчитается; иначе кэш. Аналог `useMemo` в React.

### derivedStateOf

Используется, когда результат вычисляется из нескольких state'ов, но **меняется реже**:

```kotlin
val listState = rememberLazyListState()
val showScrollToTop by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 5 }
}
```

Без `derivedStateOf` — рекомпозиция на **каждый scroll pixel** (firstVisibleItemIndex меняется на каждый item, но мы хотим только true/false). С `derivedStateOf` — рекомпозиция только при пересечении границы.

### mutableStateListOf / mutableStateMapOf

Observable-обёртки над списком/мапой:

```kotlin
val items = remember { mutableStateListOf<Item>() }
items += newItem        // triggers recomposition
```

Альтернатива: `mutableStateOf(persistentListOf<Item>())` + переприсваивание (immutable list).

### produceState

Превращает suspend/Flow в `State`:

```kotlin
val user by produceState<User?>(initialValue = null, userId) {
    value = repo.fetchUser(userId)
}
```

При смене `userId` старый блок отменяется → старт нового.

### collectAsState / collectAsStateWithLifecycle

```kotlin
val ui by viewModel.uiState.collectAsStateWithLifecycle()
```

`collectAsStateWithLifecycle` останавливает сбор при `STOPPED` (рекомендуется на Android — не жжёт батарею в фоне). Обычный `collectAsState` собирает всё время composition.

### State hoisting

Pattern: state «поднимается» наверх, composable принимает значение + callback. Делает composable stateless и тестируемым.

```kotlin
@Composable
fun MyTextField(value: String, onValueChange: (String) -> Unit) { ... }

// caller
var text by rememberSaveable { mutableStateOf("") }
MyTextField(text, { text = it })
```

### State holders / ViewModel

- **Простой state** → внутри composable (`remember`).
- **UI state на экран** → state holder (plain class) или `ViewModel`.
- **State, переживающий screen** → `ViewModel` + `SavedStateHandle`.

## Подводные камни

- `mutableStateOf(...)` без `remember` → state создаётся заново на каждой recomposition (баг: значение всегда сбрасывается).
- `rememberSaveable` НЕ работает для несериализуемых типов — нужен `Saver`.
- `derivedStateOf` дорог; не использовать там, где вычисление и так дешёвое и зависит от 1 state.
- Передача `MutableState<T>` вниз → coupling; лучше `value + onValueChange`.
- `collectAsState` без lifecycle → батарея и фоновая работа.

## Связанные темы

- [[q-01-compose-mental-model]]
- [[q-03-compose-side-effects]]
- [[../02-Android-Core/q-09-process-death]]

## Follow-up

- Когда `derivedStateOf` реально нужен, а когда — лишняя сложность?
- В чём разница между `collectAsState` и `collectAsStateWithLifecycle`?
- Как сохранить кастомный объект через `rememberSaveable`?
