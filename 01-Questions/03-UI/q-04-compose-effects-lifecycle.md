---
type: question
category: ui
difficulty: senior
tags: [compose, lifecycle, effect-scope]
created: 2026-06-16
last_reviewed: 2026-06-16
times_failed: 0
times_passed: 1
status: learning
---

# Compose: lifecycle composition vs lifecycle Android, repeatOnLifecycle

## Краткий ответ (TL;DR)

Composition имеет **свой лайфтайм** (от `enter composition` до `leave composition`), не равный Android-`Lifecycle`. На UI thread они синхронизированы (composition уходит при `onDestroy`), но `LaunchedEffect` НЕ автоматически стопит работу при `onStop` — фактически собирает Flow даже в фоне. Решение: `collectAsStateWithLifecycle` или `repeatOnLifecycle`.

## Развёрнутый ответ

### Два лайфтайма

| Лайфтайм | Когда «жив» |
|----------|-------------|
| Composition | composable в дереве (от первого вызова до удаления) |
| Lifecycle (Android) | `CREATED` ↔ `STARTED` ↔ `RESUMED` |

На обычной Activity-Compose интеграции composition уходит только при `onDestroy`, но Activity может быть `STOPPED` (свёрнута) — composition ещё жив, `LaunchedEffect` продолжает работать.

### Почему это плохо

```kotlin
LaunchedEffect(Unit) {
    locationFlow.collect { update(it) }   // ← собирает в фоне → батарея
}
```

### collectAsStateWithLifecycle

```kotlin
val ui by viewModel.uiState.collectAsStateWithLifecycle(
    minActiveState = Lifecycle.State.STARTED  // default
)
```

Под капотом: `flow.flowWithLifecycle(lifecycle, minActiveState).collectAsState(initial)`. При `STOPPED` коллектор останавливается, при `STARTED` — заново подписывается.

### repeatOnLifecycle

Низкоуровневый аналог:

```kotlin
LaunchedEffect(lifecycleOwner) {
    lifecycleOwner.lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
        locationFlow.collect { update(it) }
    }
}
```

При уходе в `STOPPED` блок отменяется, при возврате — запускается заново.

### LocalLifecycleOwner

```kotlin
val owner = LocalLifecycleOwner.current
```

Получаем актуальный `LifecycleOwner` — обычно Activity или Fragment. В обычной Compose-Activity лайфтайм совпадает; во фрагменте — это `viewLifecycleOwner` (важно для cleanup).

### Compose в ViewPager / LazyColumn

В `LazyColumn` элементы вне видимости **выходят из composition** → их `LaunchedEffect` отменяются. Если нужно сохранить state — `rememberSaveable` или поднять в ViewModel.

В `HorizontalPager` все страницы могут быть в composition (зависит от `beyondBoundsPageCount`).

### DisposableEffect для системных коллбэков

```kotlin
DisposableEffect(lifecycleOwner) {
    val obs = LifecycleEventObserver { _, e ->
        if (e == Lifecycle.Event.ON_RESUME) trackScreenView()
    }
    lifecycleOwner.lifecycle.addObserver(obs)
    onDispose { lifecycleOwner.lifecycle.removeObserver(obs) }
}
```

### LocalConfiguration / LocalContext

Меняются при configuration change → composition recomposes автоматически.

```kotlin
val conf = LocalConfiguration.current
if (conf.orientation == ORIENTATION_LANDSCAPE) { ... }
```

## Подводные камни

- `collectAsState` без lifecycle → утечка батареи / лишняя нагрузка в фоне.
- `LaunchedEffect(Unit)` для one-shot action (snackbar) → при rotation вызовется снова, дубль.
- `repeatOnLifecycle` дороже одного `collect` (re-subscribes); для редких StateFlow это ок, для частых — overhead.
- В LazyColumn `LaunchedEffect` элементов отменяется при scroll out — не использовать для важных one-time эффектов.
- Compose Activity без `setContent` не имеет composition — `LocalLifecycleOwner` всё равно работает, но composables нет.

## Связанные темы

- [[q-03-compose-side-effects]]
- [[../02-Android-Core/q-01-activity-lifecycle]]
- [[../05-Concurrency/q-flow]]

## Follow-up

- Чем `collectAsState` отличается от `collectAsStateWithLifecycle`?
- Когда composition умирает, а Android lifecycle — нет?
- Зачем `repeatOnLifecycle` если есть `LaunchedEffect`?
