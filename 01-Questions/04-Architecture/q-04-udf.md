---
type: question
category: architecture
difficulty: senior
tags: [udf, unidirectional-data-flow, state]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Unidirectional Data Flow (UDF), Single Source of Truth

## Краткий ответ (TL;DR)

**UDF**: данные идут в **одну сторону** — events (input) идут наверх (UI → VM → Domain), state идёт вниз (Domain → VM → UI). Это исключает race conditions и непредсказуемые мутации. **Single Source of Truth (SSoT)** — каждое данное имеет **один** authoritative источник; UI отображает копию, никогда не мутирует напрямую.

## Развёрнутый ответ

### Поток данных

```
                  ┌────────────────────┐
   Event/Intent → │ ViewModel (state)  │ → StateFlow → UI
   (user input)   │   ↓ ↑              │
                  │  UseCase           │
                  │   ↓ ↑              │
                  │  Repository (SSoT) │ ← DB/Cache (single source)
                  │   ↓ ↑              │
                  │  Remote / Local    │
                  └────────────────────┘
```

UI **никогда** не мутирует state напрямую. Только эмитит events.

### Bad: bidirectional

```kotlin
// ❌ UI меняет field на ViewModel, ViewModel меняет field обратно
vm.userName = "Jane"           // UI пишет
vm.observeUserName = { ... }   // и читает обратно

// race: кто последний пишет
```

### Good: UDF

```kotlin
class FormVm : ViewModel() {
    private val _state = MutableStateFlow(FormState())
    val state: StateFlow<FormState> = _state.asStateFlow()

    fun onEvent(event: FormEvent) {
        _state.update { current ->
            when (event) {
                is FormEvent.NameChanged -> current.copy(name = event.value)
                is FormEvent.Submit -> current.copy(submitting = true)
            }
        }
    }
}

@Composable
fun FormScreen(vm: FormVm) {
    val state by vm.state.collectAsStateWithLifecycle()
    TextField(value = state.name, onValueChange = { vm.onEvent(FormEvent.NameChanged(it)) })
}
```

UI рисует state, эмитит events. `_state.update` атомарно через `MutableStateFlow`.

### Single Source of Truth

Сценарий: данные о пользователе.

**Bad**: UI кеширует, Repository кеширует, Network тоже хранит. При обновлении — кому верить?

**Good**: SSoT = **Room DB**. Все читают из неё (Flow), Network обновляет её:

```kotlin
class UserRepository(
    private val dao: UserDao,
    private val api: UserApi
) {
    fun observeUser(id: Long): Flow<User> = dao.observe(id).map(::toDomain)

    suspend fun refresh(id: Long) {
        val dto = api.fetch(id)
        dao.insert(dto.toEntity())   // UI автоматически получит через Flow
    }
}
```

Преимущества:
- UI всегда консистентна — одно дерево истины.
- Offline-first работает: рисуем из DB, фоном обновляем.
- Нет «show stale data while fetching»: новые данные → DB → Flow → recomposition.

### NetworkBoundResource pattern

Старый шаблон (Google sample):

```kotlin
fun <T> networkBoundResource(
    query: () -> Flow<T>,
    fetch: suspend () -> T,
    saveFetchResult: suspend (T) -> Unit,
    shouldFetch: (T) -> Boolean = { true }
): Flow<Resource<T>> = flow { ... }
```

Сегодня заменяется Store/StoreX или ручным комбо `dao.observe() + api.fetch() → dao.insert()`.

### UI Events vs UI State

```kotlin
data class UiState(val items: List<Item>, val isLoading: Boolean)  // state — stable, наблюдаемое

sealed interface UiEvent {
    data object NavigateBack : UiEvent
    data class ShowSnackbar(val msg: String) : UiEvent
}

private val _events = Channel<UiEvent>(Channel.BUFFERED)
val events: Flow<UiEvent> = _events.receiveAsFlow()
```

State = persistent observable, events = one-shot. См. [[q-05-side-effects]].

### Compose + UDF

Compose **естественно** UDF: composable читает state, эмитит lambdas:

```kotlin
@Composable
fun Counter(state: Int, onIncrement: () -> Unit) {
    Button(onClick = onIncrement) { Text("$state") }
}
```

`state hoisting` — поднимаем state наверх для контроля.

### Тестирование

UDF легко тестируется:

```kotlin
@Test fun `submit transitions to loading`() = runTest {
    val vm = FormVm()
    vm.state.test {
        assertEquals(FormState(), awaitItem())
        vm.onEvent(FormEvent.Submit)
        assertEquals(FormState(submitting = true), awaitItem())
    }
}
```

`turbine` для test flow.

## Подводные камни

- `MutableStateFlow.value = x.copy(y = z)` без `update {}` — race при concurrent calls. Использовать `_state.update { it.copy(...) }`.
- Несколько ViewModels с разными копиями одних данных → out of sync. SSoT в Repository.
- TextField в Compose: `var text by remember { mutableStateOf("") }` — локальный state ok, но при rotation сбросится. Используй `rememberSaveable` или поднимай в VM.
- One-shot event как state (`showSnackbar: Boolean`) → snackbar показывается снова при rotation. Использовать Channel/SharedFlow.
- UI напрямую обращается к Repository, минуя ViewModel → теряется единая точка трансформации.

## Связанные темы

- [[q-01-mvc-mvp-mvvm-mvi]]
- [[q-05-side-effects]]
- [[q-09-repository-pattern]]

## Follow-up

- Что такое Single Source of Truth и где он живёт обычно?
- Почему UDF проще тестировать?
- Чем UI event отличается от UI state в плане хранения?
