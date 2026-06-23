---
type: question
category: concurrency
difficulty: senior
tags: [coroutines, testing, runtest]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Тестирование корутин: runTest, TestDispatcher, virtual time

## Краткий ответ (TL;DR)

`runTest { }` — корутиновый аналог `runBlocking` для тестов. Использует **virtual time**: `delay(1000)` пропускается мгновенно. `TestDispatcher` имеет два режима: `StandardTestDispatcher` (явный advance через `runCurrent`/`advanceTimeBy`) и `UnconfinedTestDispatcher` (eager выполнение). `Dispatchers.setMain(testDispatcher)` подменяет Main в тестах. Для StateFlow/SharedFlow — `Turbine` (`flow.test { awaitItem() }`).

## Развёрнутый ответ

### runTest базово

```kotlin
@Test
fun loadUser_returnsUser() = runTest {
    val repo = FakeUserRepo()
    val vm = UserVm(repo)

    vm.load(42)
    runCurrent()   // выполнить запланированные на сейчас корутины

    assertEquals(UiState.Success(repo.fakeUser), vm.state.value)
}
```

`runTest` ждёт завершения всех корутин launched в test scope.

### Virtual time

```kotlin
@Test
fun debouncedSearch() = runTest {
    val flow = MutableStateFlow("")
    val results = mutableListOf<String>()

    val job = launch {
        flow.debounce(300).collect { results += it }
    }

    flow.value = "a"
    flow.value = "ab"
    flow.value = "abc"
    advanceTimeBy(400)
    runCurrent()

    assertEquals(listOf("abc"), results)
    job.cancel()
}
```

`delay(300)` пропускается виртуально, тест выполняется мгновенно.

### TestDispatcher

#### StandardTestDispatcher

```kotlin
val dispatcher = StandardTestDispatcher()
```

- Корутины НЕ выполняются автоматически.
- Нужно явно `advanceTimeBy`, `runCurrent`, `advanceUntilIdle`.
- Более предсказуем для проверки промежуточных состояний.

#### UnconfinedTestDispatcher

```kotlin
val dispatcher = UnconfinedTestDispatcher()
```

- Корутины выполняются eagerly (как `Dispatchers.Unconfined`).
- Удобно для простых тестов без проверки timing.
- Не нужен `runCurrent`.

`runTest` по умолчанию использует `StandardTestDispatcher`.

### MainCoroutineRule

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class MainDispatcherRule(
    val testDispatcher: TestDispatcher = UnconfinedTestDispatcher()
) : TestWatcher() {
    override fun starting(description: Description) {
        Dispatchers.setMain(testDispatcher)
    }
    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}

class UserVmTest {
    @get:Rule val mainRule = MainDispatcherRule()

    @Test fun load() = runTest {
        val vm = UserVm(FakeRepo())
        vm.load()
        // viewModelScope.launch(Main) теперь работает
    }
}
```

Без подмены `Dispatchers.Main` — `MissingMainCoroutineDispatcherException`.

### Inject Dispatcher через DI

```kotlin
class UserRepo @Inject constructor(
    private val api: UserApi,
    @IoDispatcher private val io: CoroutineDispatcher
) {
    suspend fun load(): User = withContext(io) { api.fetch() }
}

// Тест:
val repo = UserRepo(fakeApi, StandardTestDispatcher())
```

Можно подменить любой dispatcher одним TestDispatcher для контроля timing.

### Turbine — тесты Flow

```kotlin
import app.cash.turbine.test

@Test fun stateProgression() = runTest {
    val vm = UserVm(FakeRepo())
    vm.state.test {
        assertEquals(UiState.Loading, awaitItem())
        vm.load()
        assertEquals(UiState.Success(...), awaitItem())
        cancelAndIgnoreRemainingEvents()
    }
}
```

Turbine упрощает тестирование Flow с timing'ом.

### Тестирование SharedFlow events

```kotlin
@Test fun navigationFlow() = runTest {
    val vm = MainVm()
    vm.navigation.test {
        vm.onProfileClick()
        assertEquals(NavCommand.ToProfile, awaitItem())
    }
}
```

### advanceTimeBy / runCurrent / advanceUntilIdle

```kotlin
runCurrent()           // выполнить корутины, готовые сейчас
advanceTimeBy(1000)    // пропустить 1 сек virtual time
advanceUntilIdle()     // выполнить всё пока есть запланированные
currentTime            // текущее virtual time в ms
```

### testScheduler

```kotlin
@Test fun scheduling() = runTest {
    val scheduler = testScheduler
    println(scheduler.currentTime)   // 0
    delay(1000)
    println(scheduler.currentTime)   // 1000
}
```

### Тестирование retry

```kotlin
@Test fun retryWithBackoff() = runTest {
    var attempts = 0
    val result = retryWithBackoff(times = 3, initialDelay = 1000) {
        attempts++
        if (attempts < 3) throw IOException()
        "success"
    }
    assertEquals(3, attempts)
    assertEquals("success", result)
    // delay(1000) + delay(2000) пропущены виртуально
}
```

### Тестирование Channel

```kotlin
@Test fun channelEvents() = runTest {
    val channel = Channel<String>()
    launch { channel.send("hi") }
    assertEquals("hi", channel.receive())
}
```

### Тестирование с реальным dispatcher

Иногда нужно (например, тест Room с реальной БД):

```kotlin
@Test fun realDb() = runBlocking {
    val db = Room.inMemoryDatabaseBuilder(...).build()
    db.userDao().insert(user)
    val loaded = db.userDao().get(user.id)
    assertEquals(user, loaded)
}
```

`runBlocking` — для integration тестов с реальной асинхронностью.

### TestScope.backgroundScope

```kotlin
@Test fun longRunning() = runTest {
    val data = MutableSharedFlow<Int>()

    val results = mutableListOf<Int>()
    backgroundScope.launch {
        data.collect { results += it }
    }

    data.emit(1)
    data.emit(2)
    runCurrent()
    assertEquals(listOf(1, 2), results)
    // backgroundScope автоматически отменяется в конце runTest
}
```

`backgroundScope` для never-ending корутин (StateFlow collect), чтобы runTest не повис.

## Подводные камни

- Забыл `Dispatchers.setMain` → `MissingMainCoroutineDispatcherException` при viewModelScope.
- `runTest` со StateFlow collect без `backgroundScope` → бесконечный collect, тест не завершается.
- Перепутал `runBlocking` и `runTest` — runBlocking использует реальное время, делает тесты медленными.
- `UnconfinedTestDispatcher` маскирует проблемы с порядком (всё eager) — `StandardTestDispatcher` строже.
- `Dispatchers.IO` напрямую вместо инжекта → нельзя подменить на TestDispatcher.
- `delay` после `advanceUntilIdle` — virtual time может быть в будущем, поведение неинтуитивно.

## Связанные темы

- [[q-03-dispatchers]]
- [[../12-Testing/q-coroutine-testing]]

## Follow-up

- Что такое virtual time в `runTest`?
- Чем `StandardTestDispatcher` отличается от `UnconfinedTestDispatcher`?
- Зачем `Dispatchers.setMain` в тестах?
