---
type: question
category: kotlin
difficulty: senior
tags: [kotlin, coroutines, context]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# `CoroutineScope`, `CoroutineContext`, `Job`, `Dispatchers` — кто за что отвечает?

## Краткий ответ (TL;DR)

- **`CoroutineContext`** — индексированная мапа элементов (`Job`, `Dispatcher`, `Name`, `ExceptionHandler`).
- **`CoroutineScope`** — обёртка над контекстом, привязывает корутины к жизненному циклу через `Job`.
- **`Job`** — handle: `cancel/join/children/state`, реализует структурную конкурентность.
- **`Dispatcher`** — куда планировать resume (`Main`, `Default`, `IO`, `Unconfined`).

## Развёрнутый ответ

### CoroutineContext

```kotlin
val ctx: CoroutineContext = Dispatchers.IO + Job() + CoroutineName("loader")
```

Элементы складываются оператором `+`. Каждый ключуется своим `CoroutineContext.Key`. Можно достать: `ctx[Job]`, `ctx[CoroutineDispatcher.Key]`.

### CoroutineScope

```kotlin
class MyScope : CoroutineScope {
    override val coroutineContext = Dispatchers.Main + SupervisorJob()
}
```

Готовые скоупы:
- `viewModelScope` — отменяется в `onCleared`, Dispatchers.Main.immediate + SupervisorJob.
- `lifecycleScope` — привязан к `Lifecycle`, отменяется при DESTROYED.
- `GlobalScope` — **антипаттерн**: не привязан ни к чему, не отменяется.

### Job — структурная конкурентность

```
Parent Job
 ├─ Child 1 (Job)
 │   └─ Grandchild
 └─ Child 2 (Job)
```

Правила:
1. Cancel родителя → отменяет всех детей.
2. Cancel ребёнка → НЕ влияет на родителя/братьев (если просто `Job`).
3. Исключение в ребёнке → отменяет родителя → отменяет братьев. **Кроме `SupervisorJob`**.
4. Родитель `join`-ится со всеми детьми → не завершается, пока дети живы.

### SupervisorJob vs Job

```kotlin
val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
scope.launch { throw IOException() }   // упала только эта корутина
scope.launch { /* живёт */ }
```

С обычным `Job` упали бы обе и сам scope.

### Dispatchers

| Dispatcher | Когда | Threads |
|------------|-------|---------|
| `Main` | UI работа на Android | UI thread |
| `Main.immediate` | если уже в Main — без post | — |
| `Default` | CPU-bound (parsing, math, sorting) | пул = max(2, cores) |
| `IO` | блокирующий IO (диск, сеть с blocking API) | элемент пула, расширяется до 64 |
| `Unconfined` | продолжает на текущем потоке (для тестов/специальных случаев) | — |

`Default` и `IO` шарят thread pool, но `IO` может пухнуть до 64 при необходимости.

### withContext

```kotlin
withContext(Dispatchers.IO) { blocking() }
```

Переключает контекст до конца блока, потом возвращается обратно. Это suspend-вызов: пока blocking() выполняется, поток-вызыватель освобождён.

## Подводные камни

- `GlobalScope.launch` — утечки, неконтролируемые задачи. Только в редких legitimate-случаях.
- Создание корутины внутри корутины без `coroutineScope { }` — потеря структурной конкурентности.
- `CoroutineScope(Dispatchers.Main)` без `Job` → ок, но cancel не отменяет других (внутри generate-ся SupervisorJob? Нет, обычный Job).
- `Dispatchers.IO` ≠ async non-blocking. Это thread-pool под blocking API. Для real async (suspend Retrofit, Room) `withContext(IO)` не нужен.

## Связанные темы

- [[q-10-coroutines-basics]]
- [[q-12-cancellation-exceptions]]

## Follow-up

- Когда `Main.immediate` лучше `Main`?
- Если корутина запущена в `IO` и внутри `withContext(Default) { }`, что происходит с потоком?
- Почему `GlobalScope` помечен `@DelicateCoroutinesApi`?
