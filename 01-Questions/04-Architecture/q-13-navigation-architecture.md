---
type: question
category: architecture
difficulty: senior
tags: [navigation, modularization, deeplinks]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Navigation в многомодульном приложении: контракты, deeplinks

## Краткий ответ (TL;DR)

В мультимодульном проекте feature-модули **не зависят друг от друга** — нужно решать как навигировать между ними. Подходы: **api-модуль** с route-константами, **central NavGraph** в `:app` со всеми routes, **navigation contract interface** (DI-инджекшен `Navigator`), **deeplink router**. С Compose Navigation 2.7+ type-safe routes (kotlinx.serialization).

## Развёрнутый ответ

### Проблема

```
:feature-cart  ─→ хочет открыть → :feature-checkout
:feature-cart  ─→ не может зависеть от :feature-checkout (cyclic risk)
```

### Подход 1: api-модули

```
:feature-checkout-api      // route + arguments
:feature-checkout-impl     // UI, VM (зависит от api)
:feature-cart-impl         // зависит от :feature-checkout-api для navigate
:app                       // зависит от всех -impl, регистрирует NavGraph
```

```kotlin
// :feature-checkout-api
@Serializable
data class CheckoutRoute(val cartId: Long)

// :feature-checkout-impl
fun NavGraphBuilder.checkoutScreen(onBack: () -> Unit) {
    composable<CheckoutRoute> { entry ->
        val route: CheckoutRoute = entry.toRoute()
        CheckoutScreen(cartId = route.cartId, onBack = onBack)
    }
}

// :feature-cart-impl
navController.navigate(CheckoutRoute(cartId = 42))

// :app
NavHost(...) {
    cartScreen(onCheckout = { id -> navController.navigate(CheckoutRoute(id)) })
    checkoutScreen(onBack = { navController.popBackStack() })
}
```

### Подход 2: Navigator interface

```kotlin
// :core-navigation
interface AppNavigator {
    fun toCheckout(cartId: Long)
    fun toProfile(userId: Long)
    fun back()
}

class AppNavigatorImpl(private val navController: NavController) : AppNavigator {
    override fun toCheckout(cartId: Long) = navController.navigate(...)
    // ...
}
```

VM получает `Navigator` через DI, не имеет прямой связи с `NavController`. Проблема: NavController привязан к Composable lifecycle → нужен careful scoping.

### Подход 3: Navigation commands (event-based)

```kotlin
// VM:
sealed interface NavCommand {
    data class ToCheckout(val cartId: Long) : NavCommand
    data object Back : NavCommand
}

private val _nav = Channel<NavCommand>()
val nav = _nav.receiveAsFlow()

fun onCheckout() = viewModelScope.launch {
    _nav.send(NavCommand.ToCheckout(cartId))
}

// Composable:
LaunchedEffect(Unit) {
    vm.nav.collect { cmd ->
        when (cmd) {
            is NavCommand.ToCheckout -> navController.navigate(CheckoutRoute(cmd.cartId))
            NavCommand.Back -> navController.popBackStack()
        }
    }
}
```

VM не знает о NavController, но знает что бывают такие команды. Маппинг команд в навигацию — в Compose.

### Type-safe routes (Compose 2.7+)

```kotlin
@Serializable
data class ProductDetail(val id: Long, val source: String? = null)

NavHost(navController, startDestination = Home) {
    composable<Home> { HomeScreen() }
    composable<ProductDetail> { entry ->
        val args: ProductDetail = entry.toRoute()
        ProductDetailScreen(args.id, args.source)
    }
}

navController.navigate(ProductDetail(id = 42))
```

Без строковых routes, kotlinx.serialization сериализует/десериализует args автоматически.

### Deep links в multi-module

```kotlin
// :feature-checkout-impl
composable<CheckoutRoute>(
    deepLinks = listOf(navDeepLink { uriPattern = "https://app.com/checkout/{cartId}" })
) { ... }
```

Каждый feature-модуль регистрирует свои deep links. `:app` собирает их в один граф через extension функции (`NavGraphBuilder.checkoutScreen()`).

### Nested graphs

```kotlin
@Serializable object CheckoutGraph

navigation<CheckoutGraph>(startDestination = ShippingStep) {
    composable<ShippingStep> { ... }
    composable<PaymentStep> { ... }
    composable<ConfirmStep> { ... }
}
```

ViewModel scoped на nested graph (см. [[q-02-viewmodel]]).

### Single Activity Architecture

Современная рекомендация:
- **Одна Activity** = host для NavController.
- Все экраны — Composable / Fragment в одной Activity.
- Преимущества: shared ViewModel scope, easier navigation, transitions free.

Multi-activity оправдано: completely separate flows (camera/document editor), legacy.

### Compose Navigation vs Decompose vs Voyager

- **Compose Navigation** — официальная Google, type-safe, deep links, integration with ViewModel.
- **Decompose** (Arkivanov) — для KMP, component-based, без AndroidX.
- **Voyager** — лёгкая альтернатива, screen-based, ViewModel-free.

Для production Android — Compose Navigation стандарт.

## Подводные камни

- API-модуль с реализацией внутри (`composable<X>{ ScreenX() }` в api-модуле) → impl утёк в api. Только определения routes.
- Navigator interface с захваченным NavController → утечка после Composable выхода. Использовать `DisposableEffect` или event-based.
- Deep link `uriPattern` с path-параметрами должен совпадать с args типов (`Long` → `{id}` парсится как Long).
- nested graph без `startDestination` — runtime crash.
- Compose Navigation type-safe routes требуют `kotlinx-serialization` plugin и `@Serializable`.

## Связанные темы

- [[q-08-modularization]]
- [[../02-Android-Core/q-18-deeplinks]]
- [[../03-UI/q-11-compose-navigation]]

## Follow-up

- Как навигировать между двумя feature-модулями без cyclic dependency?
- Зачем выделять `:feature-x-api` модуль?
- Single Activity vs multi-Activity — что выбрать сегодня?
