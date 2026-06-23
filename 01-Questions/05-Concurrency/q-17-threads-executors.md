---
type: question
category: concurrency
difficulty: middle
tags: [thread, executor, threadpool, java]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Threads, Executors, ThreadPool — основы JVM concurrency

## Краткий ответ (TL;DR)

`Thread` — низкоуровневая абстракция OS-потока, создавать вручную дорого. `Executor` / `ExecutorService` — пул потоков, переиспользование. Виды: `newSingleThreadExecutor`, `newFixedThreadPool(n)`, `newCachedThreadPool` (динамический), `newScheduledThreadPool` (delay/periodic), `ForkJoinPool` (work-stealing). Корутины — поверх Executor (Dispatchers.IO = unbounded pool, Default = ForkJoin). Java Virtual Threads (JDK 21) — лёгкие потоки, но в Android пока нет.

## Развёрнутый ответ

### Thread

```java
Thread t = new Thread(() -> {
    System.out.println("hello from " + Thread.currentThread().getName());
});
t.start();
t.join();   // ждать завершения
```

Дорого: ~1 MB stack, OS resource. Не создавать тысячи.

### ExecutorService

```java
ExecutorService executor = Executors.newFixedThreadPool(4);
Future<String> future = executor.submit(() -> {
    return slowComputation();
});
String result = future.get(10, TimeUnit.SECONDS);   // блокирующее ожидание
executor.shutdown();
```

### Виды пулов

| Тип | Описание |
|-----|----------|
| `newSingleThreadExecutor()` | 1 поток, sequential FIFO |
| `newFixedThreadPool(n)` | Фиксированное число потоков, очередь задач |
| `newCachedThreadPool()` | Создаёт по необходимости, переиспользует, idle timeout 60s |
| `newScheduledThreadPool(n)` | Delay / periodic выполнение |
| `newWorkStealingPool()` | ForkJoinPool с work-stealing |
| `ForkJoinPool.commonPool()` | Common pool для recursive tasks |

### ThreadPoolExecutor под капотом

```java
new ThreadPoolExecutor(
    corePoolSize,        // минимум потоков
    maximumPoolSize,     // максимум
    keepAliveTime,       // idle timeout
    timeUnit,
    workQueue,           // LinkedBlockingQueue, SynchronousQueue, ...
    threadFactory,       // named threads
    rejectedExecutionHandler  // что делать когда очередь full
);
```

### Rejection policies

- `AbortPolicy` — throw `RejectedExecutionException` (default).
- `CallerRunsPolicy` — выполняет в caller thread (backpressure).
- `DiscardPolicy` — silent drop.
- `DiscardOldestPolicy` — выбросить старейшую задачу.

### ScheduledExecutorService

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
scheduler.schedule(task, 5, TimeUnit.SECONDS);
scheduler.scheduleAtFixedRate(task, 0, 1, TimeUnit.MINUTES);
```

В Android — лучше `Handler.postDelayed` или WorkManager.

### Future / CompletableFuture

```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> fetchData(), executor)
    .thenApply(data -> parse(data))
    .thenCompose(parsed -> CompletableFuture.supplyAsync(() -> save(parsed)));

String result = future.get();
```

CompletableFuture — Java-аналог Promise / async. На Android доступен с API 24.

### Корутины поверх Executor

```kotlin
val executor = Executors.newFixedThreadPool(4)
val dispatcher = executor.asCoroutineDispatcher()

CoroutineScope(dispatcher).launch { ... }

// или из обратной стороны:
val executor = dispatcher.asExecutor()
```

`Dispatchers.IO` — настроен на 64-thread shared pool с возможностью расширения.

### Daemon threads

```java
Thread t = new Thread(task);
t.setDaemon(true);
t.start();   // JVM exit'ит даже если daemon работает
```

Не блокируют завершение процесса. Для background tasks.

### Thread priority

```java
t.setPriority(Thread.MAX_PRIORITY);   // 1..10
```

На Android — `Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND)`. Уровни вышеsem нужны редко.

### Android Looper / Handler

```java
val handlerThread = HandlerThread("worker").apply { start() }
val handler = Handler(handlerThread.looper)
handler.post { /* runs on handlerThread */ }
```

Event loop поверх Thread. Main thread всегда имеет Looper. Корутины с `Handler.asCoroutineDispatcher()`.

### Virtual Threads (JDK 21)

```java
Thread.ofVirtual().start(() -> heavyIO());
// или
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(task);
}
```

Лёгкие потоки, миллионы возможны. **На Android пока нет** (требует JDK 21 runtime). Корутины — Android-альтернатива.

### Когда Thread / Executor нужны в Kotlin/Android

- Interop с Java библиотеками (callback-based, не coroutine-friendly).
- Когда нельзя suspend (старый код).
- Когда нужен native thread (JNI).

В 99% случаев — корутины.

### ThreadLocal

```kotlin
val userIdLocal = ThreadLocal<Long>()
userIdLocal.set(42)
println(userIdLocal.get())   // 42 в этом потоке
```

В корутинах — `ThreadContextElement`:

```kotlin
launch(MyThreadLocal.asContextElement("value")) { ... }
```

### synchronized vs lock vs atomic

- `synchronized(monitor) { }` — простой mutex.
- `ReentrantLock` — больше features (tryLock, condition).
- `AtomicInteger`, `AtomicReference` — lock-free для simple operations.
- `ConcurrentHashMap`, `CopyOnWriteArrayList` — concurrent collections.

См. [[q-18-mutex-atomic]].

### Pitfall: создание threads на каждый task

```java
// ❌
new Thread(() -> uploadFile()).start();   // в цикле — десятки threads
```

Используй pool.

### Pitfall: shutdown executor

```java
ExecutorService es = Executors.newFixedThreadPool(2);
// ... use ...
es.shutdown();             // gracefully — ждёт текущие task
es.awaitTermination(10, TimeUnit.SECONDS);
es.shutdownNow();          // прерывает running tasks (interrupt)
```

Без shutdown — потоки висят, утечки.

### Performance threadpool sizing

- CPU-bound: `n = cores + 1`.
- I/O-bound: больше, формула Little's law: `n = cores * (1 + wait_time / cpu_time)`.

Android корутины: `Dispatchers.Default` = cores, `Dispatchers.IO` = max(64, cores).

## Подводные камни

- Создание Thread на каждый task → OOM при большом числе.
- Executor не shutdown → утечка threads, JVM не exit'ит.
- `Future.get()` без timeout → deadlock потенциал.
- Unbounded `newCachedThreadPool` под нагрузкой → тысячи threads.
- ThreadLocal в корутинах без `asContextElement` → значение зависит от dispatcher thread (нестабильно).
- `synchronized(this)` в библиотечном коде → пользователь может lock тот же монитор, deadlock.
- Использование Thread напрямую вместо корутин — boilerplate, no cancellation.

## Связанные темы

- [[q-03-dispatchers]]
- [[q-18-mutex-atomic]]
- [[q-19-rx-vs-coroutines]]

## Follow-up

- Чем `newCachedThreadPool` отличается от `newFixedThreadPool`?
- Зачем Executor вместо ручного Thread?
- Что такое Virtual Threads и почему их нет в Android?
