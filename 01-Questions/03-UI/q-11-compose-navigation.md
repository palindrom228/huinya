---
type: question
category: ui
difficulty: senior
tags: [compose, navigation, deep-links]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Navigation Compose: NavHost, type-safe routes, deep links, nested graphs

## Краткий ответ (TL;DR)

`NavHost` хостит граф composable-destinations. Передача аргументов — через типизированные routes (с 2.8+ — `Serializable` data class) или через `arguments`/`navArgument`. Deep links через `navDeepLink`. Каждый backstack entry — `ViewModelStoreOwner` + `SavedStateRegistryOwner` → свой ViewModel scope. Nested graphs дают логическую группировку и shared scope.

## Развёрнутый ответ

### Базовый NavHost

```kotlin
val navController = rememberNavController()
NavHost(navController, startDestination = "home") {
    composable("home") { HomeScreen(onItem = { navController.navigate("detail/$it") }) }
    composable(
        "detail/{id}",
        arguments = listOf(navArgument("id") { type = NavType.LongType })
    ) { entry ->
        val id = entry.arguments?.getLong("id") ?: return@composable
        DetailScreen(id)
    }
}
```

### Type-safe routes (2.8+)

```kotlin
@Serializable data object Home
@Serializable data class Detail(val id: Long)

NavHost(navController, startDestination = Home) {
    composable<Home> { HomeScreen(navController::navigate) }
    composable<Detail> { entry ->
        val args: Detail = entry.toRoute()
        DetailScreen(args.id)
    }
}

navController.navigate(Detail(id = 42))
```

Преимущества: компилятор проверяет аргументы, нет string concat / parsing.

### Backstack & ViewModel scope

Каждый `NavBackStackEntry` — `ViewModelStoreOwner`:

```kotlin
composable("detail/{id}") { entry ->
    val vm: DetailVm = hiltViewModel()   // привязан к entry
    DetailScreen(vm)
}
```

При pop entry → ViewModel `onCleared`. Если экран открыт повторно — новый VM.

### Shared ViewModel в nested graph

```kotlin
navigation(startDestination = "checkout/cart", route = "checkout") {
    composable("checkout/cart") { entry ->
        val parentEntry = remember(entry) { navController.getBackStackEntry("checkout") }
        val vm: CheckoutVm = hiltViewModel(parentEntry)
        ...
    }
    composable("checkout/payment") { entry ->
        val parentEntry = remember(entry) { navController.getBackStackEntry("checkout") }
        val vm: CheckoutVm = hiltViewModel(parentEntry)
        ...
    }
}
```

Оба экрана делят один VM в рамках nested graph.

### popUpTo / launchSingleTop

```kotlin
navController.navigate("home") {
    popUpTo("login") { inclusive = true }   // выкинет login из стека
    launchSingleTop = true                   // не дублировать
}
```

`popUpTo(navController.graph.startDestinationId) { inclusive = false }` — типичный pattern для логаута / bottom nav reset.

### Deep links

```kotlin
composable(
    "detail/{id}",
    deepLinks = listOf(navDeepLink { uriPattern = "https://example.com/detail/{id}" })
) { ... }
```

В Manifest:

```xml
<intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="https" android:host="example.com" />
</intent-filter>
```

### Передача результата назад

В Compose Navigation нет встроенного `setResult` как в Fragment. Pattern:

```kotlin
// в B перед popBackStack:
navController.previousBackStackEntry?.savedStateHandle?.set("result", value)
navController.popBackStack()

// в A:
LaunchedEffect(Unit) {
    val sh = navController.currentBackStackEntry?.savedStateHandle
    sh?.getStateFlow<MyData?>("result", null)
        ?.collect { result -> if (result != null) handle(result) }
}
```

### Animation between destinations

```kotlin
composable<Detail>(
    enterTransition = { slideInHorizontally { it } },
    exitTransition = { slideOutHorizontally { -it } }
) { ... }
```

### Bottom navigation

```kotlin
Scaffold(bottomBar = {
    NavigationBar {
        bottomItems.forEach { item ->
            NavigationBarItem(
                selected = currentRoute == item.route,
                onClick = {
                    navController.navigate(item.route) {
                        popUpTo(navController.graph.startDestinationId) { saveState = true }
                        launchSingleTop = true
                        restoreState = true
                    }
                },
                icon = { Icon(item.icon, null) },
                label = { Text(item.label) }
            )
        }
    }
}) { ... }
```

`saveState`/`restoreState` сохраняют состояние таба между переключениями.

## Подводные камни

- Передавать большие объекты через arguments — нельзя, только id; объект достать из репозитория.
- `hiltViewModel()` без явного `entry` всегда привязан к текущему — для shared VM передавай `parentEntry`.
- `popBackStack(route, inclusive = true)` без проверки наличия в стеке → no-op (silent).
- Bottom nav без `saveState`/`restoreState` → при переключении табов state сбрасывается.
- Deep link с типизированными аргументами → нужен `NavType` (для кастомных типов писать `NavType<T>`).

## Связанные темы

- [[q-02-compose-state]]
- [[q-12-fragment-navigation]]
- [[../02-Android-Core/q-18-deeplinks]]

## Follow-up

- Как сделать shared ViewModel в nested graph?
- Как вернуть результат с экрана B обратно на A?
- Что делает `saveState`/`restoreState` в navigate?
