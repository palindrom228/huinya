---
type: question
category: architecture
difficulty: senior
tags: [viewmodel, savedstatehandle, scope]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# ViewModel: жизненный цикл, SavedStateHandle, scope, factory

## Краткий ответ (TL;DR)

`ViewModel` живёт в `ViewModelStore`, привязанному к scope (Activity/Fragment/NavBackStackEntry). Переживает configuration change, но **не** process death. `SavedStateHandle` (injected) сохраняет данные через process death через `onSaveInstanceState` bundle. `viewModelScope` — `CoroutineScope` с `SupervisorJob + Dispatchers.Main.immediate`, отменяется в `onCleared`.

## Развёрнутый ответ

### Lifecycle

```
Activity created → ViewModel created (через ViewModelProvider/hiltViewModel)
config change → Activity recreated, ViewModel re-attached (same instance)
finish() / system kill → ViewModel.onCleared
```

`ViewModelStore` живёт пока:
- Activity не `isFinishing` (rotation сохраняет).
- Fragment в backstack или active.
- `NavBackStackEntry` в стеке (Compose Navigation).

### Создание

```kotlin
// XML / View:
class MyFragment : Fragment() {
    private val vm: MyVm by viewModels()
    private val sharedVm: SharedVm by activityViewModels()
}

// Compose:
@Composable
fun MyScreen(vm: MyVm = viewModel())

// Hilt:
@HiltViewModel
class MyVm @Inject constructor(
    private val repo: Repo,
    private val savedState: SavedStateHandle
) : ViewModel()

@Composable
fun MyScreen(vm: MyVm = hiltViewModel())
```

### SavedStateHandle

```kotlin
class DetailVm(private val sh: SavedStateHandle) : ViewModel() {
    // Чтение nav args (типизированный)
    val id: Long = sh["id"] ?: error("no id")

    // Сохранение state
    var query: String
        get() = sh["query"] ?: ""
        set(value) { sh["query"] = value }

    // StateFlow-обёртка
    val filter: StateFlow<String> =
        sh.getStateFlow("filter", "")
}
```

Что переживает:
- Configuration change — да (как ViewModel).
- Process death — **да** (сериализуется в Bundle через `onSaveInstanceState`).

Что хранить: только primitives, Parcelable, Serializable. Не объекты с context/view.

### viewModelScope

```kotlin
viewModelScope.launch {
    val users = repo.fetch()
    _state.value = UiState.Data(users)
}
```

- Создан с `SupervisorJob` — failure одной coroutine не отменяет другие.
- Dispatcher: `Dispatchers.Main.immediate`.
- Cancel автоматически в `onCleared()`.

### onCleared

```kotlin
override fun onCleared() {
    super.onCleared()
    customResource.close()
}
```

Освобождать non-coroutine ресурсы. `viewModelScope` сам cancel'ится.

### Factory

Для кастомных параметров (если не Hilt):

```kotlin
class UserVm(private val userId: Long, private val repo: Repo) : ViewModel()

class UserVmFactory(
    private val userId: Long,
    private val repo: Repo
) : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        return UserVm(userId, repo) as T
    }
}

val vm by viewModels<UserVm> { UserVmFactory(args.id, App.repo) }
```

С Hilt + `@AssistedInject`:

```kotlin
@HiltViewModel(assistedFactory = UserVm.Factory::class)
class UserVm @AssistedInject constructor(
    @Assisted val id: Long,
    private val repo: Repo
) : ViewModel() {
    @AssistedFactory interface Factory {
        fun create(id: Long): UserVm
    }
}

// usage:
val vm = hiltViewModel<UserVm, UserVm.Factory> { factory -> factory.create(id) }
```

### Scope в Navigation

```kotlin
// shared ViewModel в nested graph
val vm = hiltViewModel<CheckoutVm>(
    remember(entry) { navController.getBackStackEntry("checkout_graph") }
)
```

VM живёт пока nested graph в стеке.

### CreationExtras (1.5+)

Современный способ создания ViewModel с runtime-параметрами:

```kotlin
val vm: UserVm = viewModel(
    factory = UserVm.Factory,
    extras = MutableCreationExtras().apply {
        set(USER_ID_KEY, 42L)
    }
)
```

## Подводные камни

- ViewModel не должна знать о Context. Если очень нужно — Application context (через `AndroidViewModel` или DI), не Activity.
- `SavedStateHandle` через process death — да, но Bundle лимит ~500 KB.
- Создание ViewModel внутри composable без `viewModel()` (просто `remember { MyVm() }`) — не привязан к store → новый instance на каждом recomposition root.
- `viewModelScope.launch(Dispatchers.IO)` — забыли переключить на Main для UI → crash при collect в Compose. Используй Main.immediate.
- Sharing ViewModel между Activity невозможно без явного service-style объекта; `activityViewModels()` работает только внутри одной Activity.

## Связанные темы

- [[q-01-mvc-mvp-mvvm-mvi]]
- [[../02-Android-Core/q-09-process-death]]
- [[q-06-hilt-dagger]]

## Follow-up

- Что переживает SavedStateHandle, чего не переживает ViewModel?
- Какой Dispatcher у `viewModelScope` по умолчанию?
- Как сделать ViewModel с runtime-аргументом через Hilt?
