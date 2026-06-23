---
type: category-index
category: concurrency
tags: [index, concurrency]
---

# 5. Concurrency

Coroutines глубоко, RxJava, Thread / Handler / Looper, structured concurrency.

## Подтемы

### Coroutines
- `suspend` под капотом (continuation-passing style, state machine)
- `CoroutineScope`, `CoroutineContext`, `Job`, `Dispatchers`
- Structured concurrency: parent–child, отмена, исключения
- `SupervisorJob`, `supervisorScope`, `coroutineScope`
- `CoroutineExceptionHandler`, `try/catch`, `CancellationException`
- `withContext`, `async`/`await`, `launch`
- Flow operators: `flatMapMerge/Concat/Latest`, `buffer`, `conflate`, `debounce`
- StateFlow vs SharedFlow vs LiveData
- `callbackFlow`, `channelFlow`, `Channel`
- Тестирование корутин (`runTest`, `TestDispatcher`)

### RxJava
- Observable / Flowable / Single / Maybe / Completable
- Hot vs Cold, Subject / Processor
- Schedulers, `subscribeOn` vs `observeOn`
- Backpressure стратегии
- Combine operators (`merge`, `zip`, `combineLatest`, `switchMap`)
- Disposable, утечки, `CompositeDisposable`
- Миграция RxJava → Coroutines/Flow

### Threads
- Thread, Handler, Looper, MessageQueue
- HandlerThread, AsyncTask (deprecated, но спрашивают)
- Executors, ThreadPool, `ForkJoinPool`
- `synchronized`, `volatile`, `AtomicXxx`, happens-before

## Вопросы

```dataview
TABLE WITHOUT ID
  file.link AS "Вопрос",
  difficulty AS "Сложность",
  status AS "Статус",
  times_failed AS "Провалов"
FROM "01-Questions/05-Concurrency"
WHERE type = "question"
SORT difficulty DESC, status ASC
```
