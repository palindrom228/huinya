---
type: question
category: concurrency
difficulty: senior
tags: [deadlock, race, livelock, starvation]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Deadlock / race condition / livelock / starvation — диагностика

## Краткий ответ (TL;DR)

**Deadlock** — два+ thread'а ждут lock'и друг друга по кругу (4 условия Coffman). **Race condition** — результат зависит от порядка выполнения (read-modify-write без atomic). **Livelock** — потоки активно работают, но прогресса нет (взаимные уступки). **Starvation** — поток никогда не получает ресурс (low priority, unfair lock). Диагностика: thread dump (`jstack` / Android Studio Profiler), Strict Mode, `-Dkotlinx.coroutines.debug`. Профилактика: lock ordering, immutable data, atomic, timeout вместо infinite wait.

## Развёрнутый ответ

### Deadlock — пример

```kotlin
val lockA = Any()
val lockB = Any()

// Thread 1
synchronized(lockA) {
    Thread.sleep(100)
    synchronized(lockB) { ... }   // ждёт lockB
}

// Thread 2
synchronized(lockB) {
    Thread.sleep(100)
    synchronized(lockA) { ... }   // ждёт lockA
}
```

Кросс-локирование → deadlock.

### 4 условия Coffman (необходимы все)

1. **Mutual exclusion** — ресурс не shareable.
2. **Hold and wait** — поток держит lock и ждёт другой.
3. **No preemption** — нельзя отобрать lock у владельца.
4. **Circular wait** — цикл ожидания A→B→A.

Уберите любое — deadlock невозможен.

### Профилактика deadlock

1. **Lock ordering** — всегда захватывай locks в одном порядке (например, по `System.identityHashCode`).

```kotlin
fun transfer(a: Account, b: Account, amount: Int) {
    val (first, second) = if (a.id < b.id) a to b else b to a
    synchronized(first) {
        synchronized(second) { ... }
    }
}
```

2. **Try-lock с timeout**:

```kotlin
if (lockA.tryLock(1, TimeUnit.SECONDS)) {
    try {
        if (lockB.tryLock(1, TimeUnit.SECONDS)) {
            try { ... } finally { lockB.unlock() }
        }
    } finally { lockA.unlock() }
}
```

3. **Один lock вместо нескольких**, где возможно.

### Coroutine deadlock примеры

**`runBlocking` на main**:

```kotlin
// ❌ DEADLOCK на main thread
runBlocking(Dispatchers.Main) {
    withContext(Dispatchers.Main) { ... }
}
```

**`Channel` send без receiver**:

```kotlin
val channel = Channel<Int>()      // rendezvous
launch { channel.send(1) }        // ждёт receiver, никого нет
```

**Mutex не reentrant**:

```kotlin
val mutex = Mutex()
suspend fun a() = mutex.withLock { b() }
suspend fun b() = mutex.withLock { /* DEADLOCK */ }
```

### Race condition — пример

```kotlin
class Cache {
    var data: String? = null
    fun get(): String {
        if (data == null) {       // Thread A читает null
            data = compute()       // Thread B уже сюда — двойная работа
        }
        return data!!
    }
}
```

Fix:

```kotlin
class Cache {
    @Volatile private var data: String? = null
    private val lock = Any()
    fun get(): String =
        data ?: synchronized(lock) {
            data ?: compute().also { data = it }
        }
}
```

Или `AtomicReference`, или `lazy { }`.

### Check-then-act race

```kotlin
// ❌
if (!map.containsKey(k)) map.put(k, v)

// ✅
map.putIfAbsent(k, v)
```

### Read-modify-write race

```kotlin
// ❌
var counter = 0
fun inc() { counter++ }   // not atomic

// ✅
val counter = AtomicInteger(0)
fun inc() = counter.incrementAndGet()
```

### Livelock

Два потока бесконечно отступают друг от друга:

```kotlin
fun work() {
    while (true) {
        if (other.busy) {
            yield()       // пропустим тебя
            continue
        }
        // ...
    }
}
```

Оба yield'ят бесконечно. Fix — random backoff / приоритет.

### Starvation

- Low-priority thread не получает CPU при высокой нагрузке.
- В unfair `ReentrantLock` — поток может никогда не получить lock, если другие постоянно его захватывают.

Fix: `ReentrantLock(fair = true)`, но медленнее.

### Memory consistency / visibility

```kotlin
// ❌ другой thread может никогда не увидеть `stop = true`
var stop = false
thread { while (!stop) { ... } }
stop = true

// ✅
@Volatile var stop = false
```

Без `@Volatile` / synchronized — JVM может кешировать значение в регистре.

### Thread dump (jstack)

```bash
jstack <pid>
# или Android: Android Studio → Run → Profiler → Threads → Capture stack
```

Ищи:
- `BLOCKED` threads с локами.
- "Found one Java-level deadlock" — JVM детектит автоматически.

### Coroutines debug

```bash
-Dkotlinx.coroutines.debug=on
```

В stack trace появятся `Coroutine boundary` маркеры. Также `DebugProbes.dumpCoroutines()` (kotlinx-coroutines-debug).

### Android Strict Mode

```kotlin
StrictMode.setThreadPolicy(
    StrictMode.ThreadPolicy.Builder()
        .detectDiskReads()
        .detectNetwork()
        .detectCustomSlowCalls()
        .penaltyLog()
        .build()
)
```

Ловит блокировки main thread, дисковые операции на UI.

### ANR — связь с deadlock

5 секунд блокировки main thread → ANR. Часто причина — deadlock или sync I/O. Loglines: `at ... waiting to lock <0x...> (a SomeLock) held by ...`.

### Диагностика: heap dump + thread dump

- `adb shell am dumpheap <pid> /sdcard/dump.hprof` → HPROF, анализ в Android Studio.
- В каталоге `/data/anr/traces.txt` — ANR thread dumps.

### Lock-free алгоритмы

CAS-based структуры (`ConcurrentLinkedQueue`, `AtomicInteger`) исключают deadlock. Спин при высокой contention — компромисс.

### Immutability как решение

Immutable data → нет races. Kotlin: `val`, `data class.copy()`, persistent collections (kotlinx.collections.immutable).

### Test для concurrency багов

```kotlin
@Test fun raceCondition() {
    val counter = Counter()
    val threads = List(100) {
        thread { repeat(1000) { counter.inc() } }
    }
    threads.forEach { it.join() }
    assertEquals(100_000, counter.get())   // упадёт без atomic
}
```

Стресс-тесты ловят races. Для systematic testing — Lincheck (JetBrains).

### Lincheck пример

```kotlin
@StressCTest
class CounterTest {
    private val c = Counter()
    @Operation fun inc() = c.inc()
    @Operation fun get() = c.get()
    @Test fun runTest() = LinChecker.check(this::class.java)
}
```

Lincheck перебирает interleaving'и.

### Pitfall чек-лист

- Все mutable shared state → защищено sync / atomic / Mutex?
- Locks захватываются всегда в одном порядке?
- Long ops не под lock'ом?
- main thread не блокируется?
- Coroutines scope cancelled corectly?

## Подводные камни

- Lock внутри callback от другого потока → inverted ordering, deadlock.
- `synchronized` + `wait/notify` без while-проверки → spurious wakeup.
- AtomicReference compound (read + modify + write) без CAS-loop → race.
- `Mutex.withLock` внутри `Mutex.withLock` (тот же mutex) → corotuine deadlock.
- `runBlocking` в main thread Android → ANR.
- Слишком гранулярные locks → производительность хуже, ошибок больше.
- "Works on my machine" — race проявляется только под нагрузкой / на multi-core / в release build (JIT).

## Связанные темы

- [[q-17-threads-executors]]
- [[q-18-mutex-atomic]]
- [[q-06-cancellation]]

## Follow-up

- 4 условия deadlock?
- Как продиагностировать deadlock в Android?
- Чем livelock отличается от deadlock?
