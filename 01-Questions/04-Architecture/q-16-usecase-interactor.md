---
type: question
category: architecture
difficulty: middle
tags: [usecase, interactor, domain]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# UseCase / Interactor: когда нужны, как оформить

## Краткий ответ (TL;DR)

**UseCase** (он же Interactor) — переиспользуемая единица бизнес-логики в Domain layer. Оформляется как класс с одним `operator fun invoke(...)`. Нужен, когда: (1) одна и та же логика повторяется в нескольких VM, (2) нужна агрегация нескольких repository, (3) сложная трансформация данных. Если UseCase = одна строка `return repo.foo()` — это **boilerplate**, лучше вызвать repo напрямую.

## Развёрнутый ответ

### Базовый шаблон

```kotlin
class GetUserUseCase @Inject constructor(
    private val userRepo: UserRepository
) {
    suspend operator fun invoke(id: Long): User = userRepo.getUser(id)
}

// Использование:
class ProfileVm @Inject constructor(
    private val getUser: GetUserUseCase
) : ViewModel() {
    fun load(id: Long) = viewModelScope.launch {
        val user = getUser(id)   // выглядит как функция
    }
}
```

`operator fun invoke` — convention в Domain layer: `getUser(42)` читается как обычный вызов функции.

### Когда UseCase действительно нужен

#### Композиция нескольких repository

```kotlin
class GetEnrichedFeedUseCase @Inject constructor(
    private val newsRepo: NewsRepository,
    private val bookmarksRepo: BookmarksRepository,
    private val userRepo: UserRepository
) {
    operator fun invoke(): Flow<List<EnrichedNews>> = combine(
        newsRepo.observeNews(),
        bookmarksRepo.observeBookmarks(),
        userRepo.observeCurrent()
    ) { news, bookmarks, user ->
        news.map { item ->
            EnrichedNews(
                item = item,
                isBookmarked = item.id in bookmarks,
                isMine = item.authorId == user.id
            )
        }
    }
}
```

Без UseCase эта логика дублировалась бы в каждом VM.

#### Сложная бизнес-логика

```kotlin
class CalculateCheckoutTotalUseCase @Inject constructor(
    private val cartRepo: CartRepository,
    private val discountRepo: DiscountRepository,
    private val shippingCalculator: ShippingCalculator
) {
    suspend operator fun invoke(cartId: Long, promoCode: String?): CheckoutTotal {
        val cart = cartRepo.get(cartId)
        val discount = promoCode?.let { discountRepo.validate(it, cart.subtotal) }
        val shipping = shippingCalculator.calculate(cart.items, cart.address)
        return CheckoutTotal(
            subtotal = cart.subtotal,
            discount = discount?.amount ?: 0,
            shipping = shipping,
            total = cart.subtotal - (discount?.amount ?: 0) + shipping
        )
    }
}
```

Бизнес-правило — отдельная единица, тестируемая в изоляции.

#### Переиспользование в нескольких VM

```kotlin
class LogoutUseCase @Inject constructor(
    private val auth: AuthRepository,
    private val analytics: AnalyticsTracker,
    private val cache: CacheCleaner
) {
    suspend operator fun invoke() {
        analytics.log("logout")
        auth.signOut()
        cache.clear()
    }
}

// SettingsVm, ProfileVm, ErrorHandlerVm — все используют один LogoutUseCase
```

### Когда UseCase не нужен

```kotlin
// ❌ Бесполезный wrapper
class GetUserUseCase(private val repo: UserRepository) {
    suspend operator fun invoke(id: Long) = repo.getUser(id)
}

// ✅ Просто вызывай repo
class ProfileVm(private val userRepo: UserRepository) : ViewModel()
```

Если UseCase ничего не добавляет к repository — это нарушение [YAGNI](https://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it). Google App Architecture Guide прямо называет Domain layer **опциональным**.

### Возврат типа

```kotlin
// Suspend для one-shot
suspend operator fun invoke(id: Long): User

// Flow для observable
operator fun invoke(): Flow<List<News>>

// Result для типизированных ошибок
suspend operator fun invoke(id: Long): Result<User>
```

### UseCase с side effects

```kotlin
class SendMessageUseCase @Inject constructor(
    private val messageRepo: MessageRepository,
    private val notifier: PushNotifier
) {
    suspend operator fun invoke(text: String, chatId: Long) {
        val message = messageRepo.send(text, chatId)
        notifier.notifyRecipients(message)
    }
}
```

UseCase = command (write) или query (read). Не смешивать без необходимости (см. CQRS).

### Тестирование UseCase

```kotlin
@Test
fun `calculates total with discount`() = runTest {
    val cart = fakeCart(subtotal = 100)
    val cartRepo = FakeCartRepository(cart)
    val discountRepo = FakeDiscountRepository(Discount(amount = 10))
    val shipping = FakeShippingCalculator(returns = 5)

    val useCase = CalculateCheckoutTotalUseCase(cartRepo, discountRepo, shipping)
    val total = useCase(cartId = 1, promoCode = "SALE10")

    assertEquals(95, total.total)
}
```

Чисто unit-тестируется, без Android, без VM.

### Naming

- `Get*UseCase` — query.
- `Update*UseCase`, `Create*UseCase`, `Delete*UseCase` — command.
- `*Interactor` — синоним (Uncle Bob терминология).
- Без суффикса: `GetUser`, `Logout` — короче, читается как function. Иногда встречается, но без суффикса UseCase класс легко спутать с обычным.

### Где UseCase живёт

```
:domain
  ├── model/         (User, News, ...)
  ├── repository/    (interfaces)
  └── usecase/       (GetEnrichedFeedUseCase, LogoutUseCase, ...)
```

Domain layer не зависит от Android, Retrofit, Room — только Kotlin + Coroutines + (опционально) Flow.

### CQRS лёгкая версия

Разделить read и write по разным UseCase:

```kotlin
class GetCartUseCase(private val repo: CartRepository)         // read
class AddToCartUseCase(private val repo: CartRepository)        // write
class RemoveFromCartUseCase(private val repo: CartRepository)   // write
```

Лучше отдельные мелкие UseCase, чем один монстр-CartUseCase.

## Подводные камни

- UseCase-обёртка над одной строкой → boilerplate. Не делать "потому что Clean Architecture сказал".
- Конструктор с 8+ зависимостями → UseCase слишком жирный, разделить.
- UseCase делает несколько несвязанных вещей → нарушение SRP, разделить.
- Возврат `Flow` из UseCase, который **подписан** на `viewModelScope` — utility OK, но если внутри `stateIn` — scope зашит, теряется flexibility.
- UseCase, который держит mutable state → это уже не UseCase, а stateful service.

## Связанные темы

- [[q-03-clean-architecture]]
- [[q-09-repository-pattern]]
- [[q-12-app-architecture-guide]]

## Follow-up

- Когда UseCase оправдан, а когда нет?
- Зачем `operator fun invoke` в UseCase?
- Чем UseCase отличается от метода Repository?
