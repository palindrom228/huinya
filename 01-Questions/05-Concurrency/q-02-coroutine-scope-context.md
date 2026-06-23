---
type: question
category: concurrency
difficulty: senior
tags: [coroutines, scope, context, job]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# CoroutineScope, CoroutineContext, Job — иерархия и lifecycle

## Краткий ответ (TL;DR)

**`CoroutineContext`** — set элементов (Job, Dispatcher, CoroutineName, ExceptionHandler), индексируется по ключу. **`CoroutineScope`** = обёртка над context (`scope.coroutineContext`), привязка корутин к lifecycle (Activity/VM). **`Job`** — handle для управления (`cancel`, `join`, `children`). Иерархия: каждая дочерняя корутина наследует context от родителя, **Job всегда новый child** parent.Job. Отмена scope → отмена всех его корутин.

## Развёрнутый ответ

### CoroutineContext

```kotlin
val ctx: CoroutineContext = Dispatchers.IO + CoroutineName("loader") + Job()
```

Элементы (все реализуют `CoroutineContext.Element`):
- `Job` — управление lifecycle.
- `CoroutineDispatcher` — на каком потоке выполнять.
- `CoroutineName` — для отладки.
- `CoroutineExceptionHandler` — необработанные исключения.

Сложение `+` — merge (последний выигрывает по ключу). Доступ:
```kotlin
val job = ctx[Job]
val dispatcher = ctx[ContinuationInterceptor]
```

### CoroutineScope

```kotlin
interface CoroutineScope {
    val coroutineContext: CoroutineContext
}
```

Тонкая обёртка — нужна для:
- Привязки корутин к lifecycle (`viewModelScope`, `lifecycleScope`).
- Запрета "глобальных" корутин (best practice — отказаться от `GlobalScope`).

```kotlin
class MyVm : ViewModel() {
    fun load() = viewModelScope.launch {  // scope привязан к ViewModel
        repo.fetch()
    }
}
// При onCleared() — scope.cancel(), все launch отменяются.
```

### Создание custom scope

```kotlin
class Repository {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)

    fun start() = scope.launch { /* ... */ }
    fun stop() = scope.cancel()
}
```

`SupervisorJob` — чтобы падение одной корутины не отменяло остальные.

### Иерархия Job

```kotlin
val parentJob = Job()
val scope = CoroutineScope(parentJob)

scope.launch {       // child1.Job, parent = parentJob
    launch {         // child1.1.Job, parent = child1
        delay(1000)
    }
}

parentJob.cancel()   // → отменяется child1 → отменяется child1.1
```

Каждый `launch`/`async` создаёт **новый Job**, parent = текущий Job из context. `coroutineContext + Job()` — отрывает от родителя (плохая практика без необходимости).

### join, await, cancel

```kotlin
val job = scope.launch { delay(1000) }
job.join()         // ждать завершения
job.cancel()       // отменить
job.cancelAndJoin()// отменить и подождать
job.isActive       // true пока не завершено / не отменено
job.children       // Sequence<Job>
```

### Готовые scope в Android

| Scope | Lifecycle | Когда отменяется |
|-------|-----------|-------------------|
| `viewModelScope` | ViewModel | `onCleared()` |
| `lifecycleScope` | LifecycleOwner | `onDestroy()` |
| `LifecycleOwner.lifecycleScope.launchWhenX` (deprecated) | — | заменено на `repeatOnLifecycle` |
| `rememberCoroutineScope()` | Composable | leaves composition |
| `GlobalScope` | приложение | никогда (избегать!) |

### CoroutineContext в практике

```kotlin
viewModelScope.launch(Dispatchers.IO + CoroutineName("fetch")) {
    val data = api.get()
    withContext(Dispatchers.Main) { _state.value = data }
}
```

- `launch(ctx)` — добавляет/переопределяет элементы.
- `withContext(ctx)` — временно меняет context для блока.

### EmptyCoroutineContext

```kotlin
val ctx = EmptyCoroutineContext + Dispatchers.IO
```

Стартовая "пустая" точка для построения context.

### Inheritance правила

При `child = parent.launch { ... }`:
1. **Job**: новый, parent = `parent.coroutineContext[Job]`.
2. **Dispatcher**: наследуется (если не переопределён).
3. **Name, ExceptionHandler**: наследуется.

```kotlin
scope(Dispatchers.IO).launch {
    // dispatcher = IO (inherited)
    launch {
        // dispatcher = IO (inherited again)
    }
    launch(Dispatchers.Default) {
        // dispatcher = Default (overridden)
    }
}
```

### Зачем структурированность

```kotlin
suspend fun loadAll() = coroutineScope {
    val a = async { loadA() }
    val b = async { loadB() }
    a.await() + b.await()
    // exit: гарантировано все children завершены
}
```

`coroutineScope` ждёт всех детей перед выходом, exception в одном — отменяет всех.

## Подводные камни

- **`GlobalScope.launch`** — корутина живёт пока процесс жив, никто не отменит → утечки.
- **`CoroutineScope(Dispatchers.IO)`** без `Job()` — внутри `coroutineContext` есть auto-created Job, но без `SupervisorJob` падение child отменяет scope.
- **`+ Job()` внутри `launch(Job())`** — отрывает корутину от родителя, structured concurrency сломан.
- **`viewModelScope` + `Dispatchers.IO` без `withContext`** — VM scope использует Main по умолчанию; long blocking I/O заблокирует UI, если не вынести.
- Сравнение Job через `==`: разные instance, всегда false.
- **Cancel не отменяет немедленно** — нужна cooperation (suspension points).

## Связанные темы

- [[q-04-structured-concurrency]]
- [[q-05-supervisor-job]]
- [[q-03-dispatchers]]

## Follow-up

- Что такое CoroutineContext и какие у него элементы?
- Чем `coroutineScope { }` отличается от `CoroutineScope(...)`?
- Зачем `viewModelScope` и когда нужен custom scope?
