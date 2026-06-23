---
type: question
category: concurrency
difficulty: senior
tags: [synchronized, mutex, atomic, concurrent]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# synchronized / Mutex / Atomic / concurrent collections

## Краткий ответ (TL;DR)

**`synchronized`** — JVM-монитор, простой mutex, блокирует thread. **`ReentrantLock`** — мощнее: `tryLock`, `Condition`, fairness. **Kotlin `Mutex`** — suspend-friendly (`mutex.withLock { }`), не блокирует поток корутины. **Atomic** (`AtomicInteger`, `AtomicReference`) — lock-free CAS-операции для single value. **Concurrent collections** (`ConcurrentHashMap`, `CopyOnWriteArrayList`) — thread-safe без external sync. В корутинах: `Mutex` > `synchronized` (последний блокирует thread → может застопорить пул). Для счётчиков — atomic, для коллекций — concurrent.

## Развёрнутый ответ

### synchronized

```kotlin
class Counter {
    private var value = 0
    fun increment() = synchronized(this) {
        value++
    }
    fun get() = synchronized(this) { value }
}
```

Под капотом — monitor enter / exit на объекте. Reentrant: тот же thread может войти повторно.

**Pitfall**: `synchronized(this)` в библиотечном коде — external код может lock тот же монитор → deadlock. Используй private object:

```kotlin
private val lock = Any()
fun safe() = synchronized(lock) { ... }
```

### ReentrantLock

```kotlin
val lock = ReentrantLock()

fun work() {
    lock.lock()
    try {
        // critical section
    } finally {
        lock.unlock()
    }
}

// или
lock.withLock { ... }
```

Дополнительно:
- `tryLock(timeout)` — попытка с таймаутом.
- `Condition` — wait/notify аналог.
- `ReentrantLock(fair = true)` — FIFO порядок.

### ReentrantReadWriteLock

```kotlin
val rwLock = ReentrantReadWriteLock()
val readLock = rwLock.readLock()
val writeLock = rwLock.writeLock()
```

Много readers одновременно, writer — эксклюзивно. Для read-heavy workload.

### Kotlin Mutex (coroutines)

```kotlin
val mutex = Mutex()
var counter = 0

suspend fun increment() = mutex.withLock {
    counter++
}
```

**Главное отличие от `synchronized`**: `Mutex.lock()` — suspend. Не блокирует поток, освобождает dispatcher для других корутин. В корутинах **всегда** Mutex, не synchronized.

### Atomic

```kotlin
val counter = AtomicInteger(0)
counter.incrementAndGet()         // ++ atomic
counter.compareAndSet(0, 1)       // CAS
counter.updateAndGet { it * 2 }   // lambda update
```

Внутри — CPU CAS-инструкция (`lock cmpxchg`). Lock-free, но spin при contention.

Виды:
- `AtomicInteger` / `AtomicLong` / `AtomicBoolean`.
- `AtomicReference<T>` — для объектов.
- `AtomicIntegerArray` — массив.
- `LongAdder` / `LongAccumulator` — лучше под high contention (striped counter).

### kotlinx.atomicfu

```kotlin
import kotlinx.atomicfu.atomic

class Counter {
    private val value = atomic(0)
    fun inc() = value.incrementAndGet()
}
```

Multiplatform. На JVM компилируется в `AtomicIntegerFieldUpdater` (без обёртки).

### Volatile

```kotlin
@Volatile var initialized = false
```

Гарантирует visibility (read всегда видит последнюю запись), **но не atomicity**. Для `boolean` флагов — OK. Для `i++` — нет, нужен atomic.

### Concurrent collections

| Коллекция | Назначение |
|-----------|------------|
| `ConcurrentHashMap<K,V>` | thread-safe map, lock striping |
| `CopyOnWriteArrayList<T>` | копия при write, fast read |
| `ConcurrentLinkedQueue<T>` | lock-free queue |
| `LinkedBlockingQueue<T>` | bounded blocking queue |
| `ArrayBlockingQueue<T>` | fixed-size blocking queue |
| `SynchronousQueue<T>` | rendezvous (0 capacity) |
| `ConcurrentSkipListMap` | sorted concurrent map |

### ConcurrentHashMap atomic ops

```kotlin
val map = ConcurrentHashMap<String, Int>()

map.computeIfAbsent("key") { computeValue(it) }   // atomic
map.compute("key") { _, v -> (v ?: 0) + 1 }       // atomic update
map.merge("key", 1) { old, new -> old + new }     // atomic merge
```

Не используй `get` → `put` отдельно — race.

### CopyOnWriteArrayList

- Write копирует весь массив (дорого для больших).
- Read — без блокировки.
- Для listeners / observers, где write редок.

### Race condition пример

```kotlin
// ❌
class Counter {
    var v = 0
    fun inc() { v++ }   // read-modify-write, не atomic
}
```

Two threads, итог потерян.

```kotlin
// ✅
class Counter {
    private val v = AtomicInteger(0)
    fun inc() = v.incrementAndGet()
}
```

### Double-checked locking (для singleton)

```kotlin
class Singleton private constructor() {
    companion object {
        @Volatile private var instance: Singleton? = null
        fun get(): Singleton =
            instance ?: synchronized(this) {
                instance ?: Singleton().also { instance = it }
            }
    }
}
```

`@Volatile` обязателен. В Kotlin проще — `object Singleton`.

### StampedLock

```kotlin
val lock = StampedLock()
val stamp = lock.tryOptimisticRead()
val value = readState()
if (!lock.validate(stamp)) {
    // fallback на read lock
}
```

Optimistic read — без локирования, проверка после. Для read-heavy с редкими writes.

### Semaphore

```kotlin
val sem = Semaphore(permits = 3)
sem.acquire()
try { /* max 3 concurrent */ }
finally { sem.release() }
```

Kotlin coroutines `kotlinx.coroutines.sync.Semaphore` — suspend-friendly.

### CountDownLatch / CyclicBarrier

- **CountDownLatch** — wait until N счётчик дойдёт до 0 (one-shot).
- **CyclicBarrier** — N threads ждут друг друга, повторяется.

### thread-safe vs thread-confined

- **Thread-safe** — можно вызывать из любого thread.
- **Thread-confined** — только один thread (UI element).
- **Immutable** — auto thread-safe.

### Performance сравнение (примерно)

```
atomic ops      ~ ns
synchronized    ~ 25 ns (no contention) / >> 1 µs (contention)
Mutex (kotlin)  ~ comparable, не блокирует thread
ConcurrentHashMap.get ~ 50 ns
```

### Контекст Android

- В коде VM — обычно `Mutex` (если есть mutable state и доступ из нескольких корутин).
- StateFlow / SharedFlow внутри thread-safe — не нужно external sync.
- Room DAO — thread-safe.
- `LiveData.setValue` — main only, `postValue` — thread-safe.

## Подводные камни

- `synchronized` в suspend-функции **блокирует поток** — потеря производительности корутин.
- `synchronized(this)` в библиотечном классе → внешний deadlock.
- `Volatile` ≠ atomic — `i++` ломается.
- CopyOnWriteArrayList с большими списками + частые writes → O(n) на каждую запись.
- `Mutex` не reentrant — повторный `withLock` в той же корутине → deadlock. Используй `Mutex(holder = this)` с осторожностью или избегай reentrant логики.
- AtomicReference compound update без CAS-loop → race.
- `ConcurrentHashMap.size()` — приблизительный (под нагрузкой).

## Связанные темы

- [[q-17-threads-executors]]
- [[q-20-deadlock-race]]
- [[q-02-coroutine-scope-context]]

## Follow-up

- Чем `Mutex` отличается от `synchronized`?
- Когда atomic, когда mutex?
- Зачем `@Volatile`?
