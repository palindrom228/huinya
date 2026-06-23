---
type: question
category: architecture
difficulty: senior
tags: [state, statehoisting, savedstate]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# State management: где хранить state, hoisting, scope

## Краткий ответ (TL;DR)

State имеет **scope**: Composable-local (`remember`), screen-level (rememberSaveable / ViewModel), navigation graph (NavBackStackEntry VM), app-level (Repository, DataStore). Правило: хранить **на минимально необходимом уровне**. Hoisting = поднимать state наверх, ребёнок получает `value + onChange`. Process death survival → SavedStateHandle / rememberSaveable / DataStore.

## Развёрнутый ответ

### Иерархия scope

```
Application scope          Repository (DataStore, DB) — app-wide
   ↓
NavGraph scope             Nested graph VM (shared between screens)
   ↓
Screen scope               ViewModel, SavedStateHandle
   ↓
Composable subtree         rememberSaveable (survives recreation/PD)
   ↓
Local                      remember (только этот recomposition)
```

### Local state с remember

```kotlin
@Composable
fun ExpandableCard(title: String) {
    var expanded by remember { mutableStateOf(false) }
    // expanded reset при unmount/rotation
}
```

Подходит для transient UI state (hover, focus, expand).

### rememberSaveable

```kotlin
var query by rememberSaveable { mutableStateOf("") }
// сохраняется через onSaveInstanceState bundle
```

Survives configuration change + process death (для primitive/Parcelable).

Custom saver:
```kotlin
val FilterSaver = Saver<Filter, List<String>>(
    save = { listOf(it.category, it.brand) },
    restore = { Filter(category = it[0], brand = it[1]) }
)
var filter by rememberSaveable(saver = FilterSaver) { mutableStateOf(Filter()) }
```

### ViewModel state

Для state, которое:
- Должно пережить rotation.
- Связано с бизнес-логикой.
- Нужно шарить между Composable subtrees.

```kotlin
class ScreenVm : ViewModel() {
    private val _state = MutableStateFlow(UiState())
    val state: StateFlow<UiState> = _state.asStateFlow()
}
```

### SavedStateHandle

Для state, которое должно пережить **process death**:

```kotlin
class SearchVm(private val saved: SavedStateHandle) : ViewModel() {
    val query: StateFlow<String> = saved.getStateFlow("query", "")
    fun setQuery(q: String) { saved["query"] = q }
}
```

Только небольшие данные (Bundle лимит ~500 KB).

### State hoisting

```kotlin
// ❌ stateful child — нельзя контролировать снаружи
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) { Text("$count") }
}

// ✅ stateless child — state поднят, контролируется родителем
@Composable
fun Counter(count: Int, onIncrement: () -> Unit) {
    Button(onClick = onIncrement) { Text("$count") }
}

@Composable
fun ScreenWithCounter(vm: ScreenVm) {
    val state by vm.state.collectAsStateWithLifecycle()
    Counter(state.count, vm::increment)
}
```

Hoisting:
- Делает компонент reusable (один и тот же `Counter` в разных контекстах).
- Облегчает preview/test.
- Single source of truth (state в VM, UI рисует).

### State holders (plain class)

Не всё нужно в ViewModel. Локальный «state holder» для сложного UI:

```kotlin
class ListPagerState(
    initialPage: Int,
    private val scope: CoroutineScope
) {
    var currentPage by mutableStateOf(initialPage)
    fun next() { scope.launch { animateTo(currentPage + 1) } }
}

@Composable
fun rememberListPagerState(initial: Int = 0): ListPagerState {
    val scope = rememberCoroutineScope()
    return remember { ListPagerState(initial, scope) }
}
```

Аналогично `LazyListState`, `ScrollState` — это plain state holders.

### App-wide state

- **DataStore (Preferences/Proto)** для settings.
- **Room** для domain data.
- **In-memory cache** для session data.

Не ViewModel — VM привязан к экрану.

### Shared state между экранами

```kotlin
// nested graph scope
val parentEntry = remember(currentEntry) {
    navController.getBackStackEntry("checkout_graph")
}
val sharedVm: CheckoutVm = hiltViewModel(parentEntry)
```

ViewModel переживает все экраны nested graph.

### CompositionLocal для app-wide values

```kotlin
val LocalUserSession = compositionLocalOf<UserSession> { error("not set") }

CompositionLocalProvider(LocalUserSession provides session) {
    AppScreens()
}

// внутри:
val session = LocalUserSession.current
```

Для **редко меняющегося** контекста (theme, user, locale). Не использовать для state, который часто меняется — recomposition всего subtree.

### Что хранить где (правила)

| Что | Где |
|-----|-----|
| Hover, focus, expand | `remember` |
| Form input (после rotation) | `rememberSaveable` |
| Loaded data, business state | `ViewModel + StateFlow` |
| Surviving process death | `SavedStateHandle` |
| Cross-screen в потоке | nested graph VM |
| Глобальные settings | DataStore |
| Domain data | Room + Repository |

## Подводные камни

- `remember` внутри `LazyColumn.item` — пересоздаётся при scroll out/in. Использовать `key`.
- `mutableStateOf` без `remember` — пересоздаётся на каждой recomposition. (Обычно `var x by remember { mutableStateOf(0) }`.)
- `rememberSaveable` для большого объекта (вся бизнес-модель) — TransactionTooLargeException при PD.
- ViewModel хранит UI-only state, которое можно было держать локально → утрата простоты.
- CompositionLocal с часто-меняющимся value → recomposition всего дерева consumers.

## Связанные темы

- [[q-02-viewmodel]]
- [[q-04-udf]]
- [[../03-UI/q-02-compose-state]]

## Follow-up

- Чем `remember` отличается от `rememberSaveable`?
- Что такое state hoisting и зачем?
- Как shared state между экранами в Compose Navigation?
