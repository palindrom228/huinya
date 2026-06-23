---
type: question
category: architecture
difficulty: senior
tags: [viewmodel, shared, eventbus]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Коммуникация ViewModel ↔ ViewModel: shared VM, EventBus, repository

## Краткий ответ (TL;DR)

VM напрямую друг другу не общаются. Способы: (1) **Shared ViewModel** на NavGraph scope (sibling fragment communication), (2) **Activity-scoped VM** (для всех fragment'ов одной Activity), (3) **Repository как mediator** (предпочтительно для cross-screen state — single source of truth), (4) **SavedStateHandle + back-navigation result** (передать данные обратно), (5) **App-wide Flow** в singleton repository. EventBus / RxBus в современном Android — антипаттерн (неявные зависимости).

## Развёрнутый ответ

### Shared ViewModel via NavGraph

```kotlin
// Compose
val parentEntry = remember(currentEntry) {
    navController.getBackStackEntry("checkout_graph")
}
val sharedVm: CheckoutVm = hiltViewModel(parentEntry)
```

```kotlin
// Fragment
val sharedVm: CheckoutVm by hiltNavGraphViewModels(R.id.checkout_graph)
```

VM создаётся на nested graph, переживает экраны внутри.

### Activity-scoped VM

```kotlin
// Fragment
val activityVm: AppVm by activityViewModels()
```

Один VM на всё время Activity, доступен из всех fragment'ов.

**Минус**: shared mutable state легко превращается в god-object. Только для очевидно activity-wide данных (текущий tab, search query всей Activity).

### Repository как mediator (recommended)

```kotlin
@Singleton
class CartRepository @Inject constructor() {
    private val _cart = MutableStateFlow(Cart.EMPTY)
    val cart: StateFlow<Cart> = _cart.asStateFlow()

    fun add(item: Item) { _cart.update { it + item } }
}

// VM1 (Catalog):
class CatalogVm @Inject constructor(private val cart: CartRepository) : ViewModel() {
    fun addToCart(item: Item) { cart.add(item) }
}

// VM2 (CartScreen):
class CartVm @Inject constructor(cart: CartRepository) : ViewModel() {
    val state = cart.cart.stateIn(...)
}
```

Преимущества:
- Single source of truth.
- Изменения автоматически видны всем подписчикам.
- Repository выживает rotation/destroy.
- Тестируемо (mock repository).

**Это основной паттерн в современной Android-архитектуре.**

### SavedStateHandle для back-navigation

```kotlin
// Compose Navigation
navController.previousBackStackEntry
    ?.savedStateHandle
    ?.set("result", selectedItem)
navController.popBackStack()

// На предыдущем экране:
val result = navController.currentBackStackEntry
    ?.savedStateHandle
    ?.getStateFlow<Item?>("result", null)
    ?.collectAsState()
```

Для передачи результата выбора (PickerScreen → MainScreen).

### App-wide event Flow

```kotlin
@Singleton
class AppEventBus @Inject constructor() {
    private val _events = MutableSharedFlow<AppEvent>(replay = 0)
    val events: SharedFlow<AppEvent> = _events.asSharedFlow()

    suspend fun emit(event: AppEvent) = _events.emit(event)
}

sealed interface AppEvent {
    data object UserLoggedOut : AppEvent
    data class CartUpdated(val count: Int) : AppEvent
}
```

VM1 emit → любая другая VM может collect. Использовать **сдержанно** — иначе превращается в global state.

### EventBus / RxBus антипаттерн

```kotlin
// ❌ Старый стиль (Greenrobot EventBus и т.п.)
EventBus.getDefault().post(LogoutEvent())
@Subscribe fun onLogout(e: LogoutEvent) { ... }
```

Почему плохо:
- Скрытые зависимости (compile-time проверки нет).
- Не понятно кто кого вызывает.
- Сложно тестировать.
- Easy спагетти.

Современная замена — Flow в shared repository или Channel.

### Pattern: VM → VM через navigation event

```kotlin
// VM1:
class FormVm : ViewModel() {
    private val _nav = Channel<NavCommand>()
    val nav = _nav.receiveAsFlow()

    fun onSubmit() = viewModelScope.launch {
        save()
        _nav.send(NavCommand.ToConfirmation(orderId))
    }
}

// UI слушает и навигирует:
LaunchedEffect(Unit) {
    vm.nav.collect { cmd -> navController.navigate(...) }
}

// VM2 (ConfirmationVm) получает orderId через savedStateHandle:
class ConfirmationVm @Inject constructor(saved: SavedStateHandle) : ViewModel() {
    val orderId = saved.toRoute<ConfirmationRoute>().orderId
}
```

Прямой передачи между VM нет — через navigation arguments.

### Когда какой подход

| Сценарий | Подход |
|----------|--------|
| Sibling fragments в одной графе | NavGraph shared VM |
| Корзина видна везде | Singleton Repository |
| Передать выбранный item обратно | SavedStateHandle (popBackStack) |
| Глобальное событие (logout) | App-wide SharedFlow |
| Все fragment одной Activity | Activity-scoped VM (умеренно) |
| Разные scope экраны (Profile ↔ Settings) | Repository |

### Antipattern: VM holds reference to другой VM

```kotlin
// ❌ никогда
class ScreenAVm @Inject constructor(private val screenBVm: ScreenBVm) : ViewModel()
```

Lifecycle разный, scope разный — гарантированные leaks/crashes.

### Pattern: Domain layer events

```kotlin
class CheckoutUseCase @Inject constructor(
    private val orderRepo: OrderRepository,
    private val analyticsRepo: AnalyticsRepository,
    private val cartRepo: CartRepository
) {
    suspend operator fun invoke(cart: Cart): Order {
        val order = orderRepo.create(cart)
        cartRepo.clear()
        analyticsRepo.track(OrderCreated(order.id))
        return order
    }
}
```

Use case в domain layer оркестрирует — VM просто вызывает.

### Pattern: hilt с qualifiers

```kotlin
@Module @InstallIn(SingletonComponent::class)
object EventModule {
    @Provides @Singleton
    fun navigationEvents() = MutableSharedFlow<NavCommand>(extraBufferCapacity = 16)
}

class VmA @Inject constructor(private val nav: MutableSharedFlow<NavCommand>) : ViewModel()
class VmB @Inject constructor(private val nav: MutableSharedFlow<NavCommand>) : ViewModel()
```

Чистый Flow вместо EventBus библиотеки.

## Подводные камни

- Activity-scoped VM как helmsman всего экрана → god object.
- Singleton repository с mutable StateFlow без `update { }` → race conditions.
- EventBus / `LocalBroadcastManager` (deprecated) для VM communication.
- VM держит reference на другой VM → leaks.
- `SavedStateHandle.set` для большой модели → TransactionTooLargeException.
- SharedFlow с `replay > 0` для one-shot events → старое событие приходит новому subscriber.

## Связанные темы

- [[q-02-viewmodel]]
- [[q-05-side-effects]]
- [[q-09-repository-pattern]]
- [[q-13-navigation-architecture]]

## Follow-up

- Как два sibling fragment'а делятся state?
- Почему EventBus — антипаттерн?
- Как передать результат назад при popBackStack?
