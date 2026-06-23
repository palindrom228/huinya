---
type: question
category: ui
difficulty: senior
tags: [compose, side-effects, coroutines]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Compose side effects: LaunchedEffect, DisposableEffect, SideEffect, rememberCoroutineScope

## Краткий ответ (TL;DR)

Composable должна быть **чистой**. Любые сайд-эффекты (запуск корутины, подписка на listener, обновление внешнего state) — внутри специальных effect API: `LaunchedEffect` (suspend на лайфтайм composition), `DisposableEffect` (с `onDispose` для cleanup), `SideEffect` (синхронно после каждой successful composition), `rememberCoroutineScope` (scope для запуска корутин из callbacks).

## Развёрнутый ответ

### LaunchedEffect

Запускает корутину при входе composition; при изменении `key` — отменяет старую и запускает новую. При выходе composition — отменяет.

```kotlin
LaunchedEffect(userId) {
    val user = repo.fetchUser(userId)
    state = user
}
```

`LaunchedEffect(Unit)` — один раз за лайфтайм этого composition (популярный паттерн one-shot init).

### DisposableEffect

Для подписок, которые надо **отписать**:

```kotlin
DisposableEffect(lifecycleOwner) {
    val observer = LifecycleEventObserver { _, event -> handle(event) }
    lifecycleOwner.lifecycle.addObserver(observer)
    onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
}
```

Также нужен `key` — при изменении старый dispose, новый init.

### SideEffect

Синхронно бежит после каждой successful composition — для публикации значения наружу из Compose:

```kotlin
SideEffect {
    analytics.setScreen("home")
}
```

НЕ для async/корутин. НЕ для setup/teardown — для этого DisposableEffect/LaunchedEffect.

### rememberCoroutineScope

Когда корутину нужно запустить из **обработчика** (например, onClick):

```kotlin
val scope = rememberCoroutineScope()
Button(onClick = {
    scope.launch { snackbar.show("Saved") }
}) { Text("Save") }
```

Scope живёт пока composition жив. При уходе → отменяется.

Не использовать в onClick `LaunchedEffect` — он привязан к composition, а onClick это event handler.

### produceState

Преобразует async источник в `State`:

```kotlin
val image by produceState<Bitmap?>(null, url) {
    value = loader.load(url)
    awaitDispose { /* cancel */ }
}
```

### snapshotFlow

Превращает чтение Compose State в `Flow`:

```kotlin
LaunchedEffect(listState) {
    snapshotFlow { listState.firstVisibleItemIndex }
        .distinctUntilChanged()
        .collect { analytics.logScroll(it) }
}
```

### derivedStateOf vs LaunchedEffect

- Вычисление синхронное → `derivedStateOf`.
- Async / suspend / side-effect → `LaunchedEffect` или `produceState`.

### Правила

1. **Не запускай корутины напрямую в теле composable** — нет лайфтайма.
2. **Не вызывай ViewModel-методы из тела composable** — будет вызываться на каждой recomposition.
3. **Используй remember-ленивые callbacks** (`onClick = remember { { ... } }`) если нужно стабилизировать.
4. **Cleanup в DisposableEffect** обязателен для системных подписок.

## Подводные камни

- `LaunchedEffect(Unit)` НЕ перезапускается даже если параметры экрана меняются — иногда хотят зависимость от args, забывают.
- Запуск VM-action в `LaunchedEffect(Unit)` без guard → при возврате на экран не сработает (composition не пересоздан).
- `SideEffect` для async → не отменяется, race.
- `rememberCoroutineScope` внутри лямбды без `remember` → scope создаётся заново.
- `snapshotFlow` без `distinctUntilChanged` → может эмитить дубликаты.

## Связанные темы

- [[q-02-compose-state]]
- [[q-04-compose-effects-lifecycle]]
- [[../05-Concurrency/q-coroutines]]

## Follow-up

- Чем `LaunchedEffect` отличается от `rememberCoroutineScope`?
- Когда обязателен `DisposableEffect` вместо `LaunchedEffect`?
- Что делает `snapshotFlow` и зачем он нужен?
