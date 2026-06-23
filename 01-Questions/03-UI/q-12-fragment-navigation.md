---
type: question
category: ui
difficulty: middle
tags: [fragment, navigation-component, safe-args]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Fragment-based Navigation Component: NavHostFragment, Safe Args, multiple back stacks

## Краткий ответ (TL;DR)

Navigation Component (XML graph + `NavHostFragment`) — официальный способ для View-приложений. **Safe Args** генерит типизированные `Directions`/`Args` классы. Multiple back stacks (с 2.4) делает Bottom Navigation корректным (каждый таб — свой стек). Глубокие ссылки и аргументы — через `<argument>` и `<deepLink>` в graph.xml.

## Развёрнутый ответ

### Граф

```xml
<navigation xmlns:android="..."
    app:startDestination="@id/homeFragment">

    <fragment android:id="@+id/homeFragment"
              android:name="com.x.HomeFragment">
        <action android:id="@+id/toDetail"
                app:destination="@id/detailFragment" />
    </fragment>

    <fragment android:id="@+id/detailFragment"
              android:name="com.x.DetailFragment">
        <argument android:name="id" app:argType="long" />
        <deepLink app:uri="https://example.com/detail/{id}" />
    </fragment>
</navigation>
```

### NavHostFragment в Activity

```xml
<androidx.fragment.app.FragmentContainerView
    android:id="@+id/nav_host"
    android:name="androidx.navigation.fragment.NavHostFragment"
    app:navGraph="@navigation/main_graph"
    app:defaultNavHost="true" />
```

### Safe Args (gradle plugin)

```kotlin
// генерится HomeFragmentDirections, DetailFragmentArgs
findNavController().navigate(
    HomeFragmentDirections.toDetail(id = 42)
)

class DetailFragment : Fragment() {
    private val args by navArgs<DetailFragmentArgs>()
    // args.id : Long
}
```

Type-safe → компилятор ловит несоответствие аргументов.

### Multiple back stacks

Bottom navigation должен сохранять историю на каждом табе. С 2.4:

```kotlin
bottomNav.setupWithNavController(navController)
// автоматически использует saveState/restoreState
```

Под капотом `NavOptions`:

```kotlin
navController.navigate(itemId) {
    popUpTo(navController.graph.startDestinationId) { saveState = true }
    launchSingleTop = true
    restoreState = true
}
```

### Передача результата

```kotlin
// в B:
findNavController().previousBackStackEntry?.savedStateHandle?.set("key", value)
findNavController().popBackStack()

// в A:
val nav = findNavController()
val sh = nav.currentBackStackEntry?.savedStateHandle
sh?.getLiveData<MyData>("key")?.observe(viewLifecycleOwner) { ... }
```

### Nested graphs

```xml
<navigation android:id="@+id/checkout_graph"
            app:startDestination="@id/cart">
    <fragment android:id="@+id/cart" .../>
    <fragment android:id="@+id/payment" .../>
</navigation>
```

Shared ViewModel:

```kotlin
private val vm by hiltNavGraphViewModels<CheckoutVm>(R.id.checkout_graph)
```

VM живёт пока nested graph в стеке.

### Deep links

```xml
<deepLink app:uri="https://example.com/detail/{id}"
          app:action="android.intent.action.VIEW"
          android:autoVerify="true" />
```

С `autoVerify` + `assetlinks.json` — App Links (см. [[../02-Android-Core/q-18-deeplinks]]).

В Activity:

```kotlin
// автоматически обработается NavHostFragment'ом, если intent matches
```

### Conditional navigation

```kotlin
navController.addOnDestinationChangedListener { _, dest, _ ->
    if (dest.id == R.id.protectedFragment && !auth.isLoggedIn) {
        navController.navigate(R.id.loginFragment)
    }
}
```

Лучше — graph-level: protected destinations в отдельном subgraph + login graph.

### Animations

```xml
<action android:id="@+id/toDetail"
        app:destination="@id/detailFragment"
        app:enterAnim="@anim/slide_in"
        app:exitAnim="@anim/slide_out" />
```

## Подводные камни

- `<fragment>` vs `<dialog>` — забыли `<dialog>` для DialogFragment → не показывается как диалог.
- Safe Args требует `apply plugin: "androidx.navigation.safeargs.kotlin"` + `Parcelable` для кастомных аргументов.
- Возврат через `popBackStack` без проверки → если стек пуст → false возврат, Activity не закроется.
- `currentBackStackEntry?.savedStateHandle` для result читай в `viewLifecycleOwner` scope (иначе утечка).
- DeepLink с не-string args → нужен правильный `argType` в graph.

## Связанные темы

- [[q-11-compose-navigation]]
- [[../02-Android-Core/q-18-deeplinks]]
- [[../02-Android-Core/q-02-fragment-lifecycle]]

## Follow-up

- Что даёт multiple back stacks (2.4+) для bottom navigation?
- Чем `navArgs<T>()` лучше `arguments?.getXxx("key")`?
- Как сделать ViewModel, общий для nested graph?
