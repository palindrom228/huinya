---
type: question
category: architecture
difficulty: senior
tags: [testing, architecture, fakes, mocks]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Архитектура тестирования: test pyramid, fakes vs mocks, screenshot tests

## Краткий ответ (TL;DR)

**Test pyramid**: много unit (быстрые, дешёвые), меньше integration (Robolectric / Compose UI), мало end-to-end (Espresso, реальный device). Для unit — **fakes** предпочтительнее mocks (фейк репозиторий с in-memory state). Для UI — **screenshot tests** (Paparazzi / Roborazzi) ловят регрессии без эмулятора. Hilt + `@HiltAndroidTest` для DI в тестах. Архитектурно: layered tests — каждый слой тестируется со своими fake'ами.

## Развёрнутый ответ

### Test pyramid

```
        ╱╲
       ╱E2╲          end-to-end (Espresso, Maestro) — мало, медленно
      ╱────╲
     ╱ Integ ╲       integration (Robolectric, Compose, Room real)
    ╱──────────╲
   ╱   Unit     ╲    unit (JVM) — много, быстро, дёшево
  ╱──────────────╲
```

Цель: основная масса логики покрыта быстрыми unit-тестами.

### Unit-тесты (JVM)

```kotlin
class UserRepositoryTest {
    private val api = FakeUserApi()
    private val dao = FakeUserDao()
    private val repo = UserRepositoryImpl(api, dao)

    @Test fun load_cachesInDao() = runTest {
        api.users = listOf(UserDto(1, "Alice"))
        repo.refresh()
        assertEquals(listOf(UserEntity(1, "Alice")), dao.allUsers())
    }
}
```

Запускаются на JVM, секунды на full suite, без эмулятора.

### Fakes vs Mocks

**Fake** — рабочая реализация для тестов:

```kotlin
class FakeUserApi : UserApi {
    var users: List<UserDto> = emptyList()
    var error: Throwable? = null

    override suspend fun fetchUsers(): List<UserDto> {
        error?.let { throw it }
        return users
    }
}
```

**Mock** (MockK / Mockito):

```kotlin
val api = mockk<UserApi>()
coEvery { api.fetchUsers() } returns listOf(...)
```

| | Fake | Mock |
|---|------|------|
| Подготовка | один раз | в каждом тесте |
| Изменения сигнатуры | пере-компиляция fake | пере-настройка mocks |
| Поведение | реальное (in-memory) | программируемое |
| Сложность | растёт с тестом | растёт со сложностью теста |

**Google рекомендует fakes** для test-only кода в production коде нет — fake тестируется через свои тесты.

### Test doubles терминология

- **Dummy** — заглушка, не используется (заполнение параметров).
- **Stub** — возвращает заранее заданные ответы.
- **Fake** — упрощённая реализация (in-memory DB).
- **Mock** — проверяет вызовы (verify count, order, args).
- **Spy** — реальный объект с подменой части методов.

### ViewModel тесты

```kotlin
@get:Rule val mainDispatcherRule = MainDispatcherRule()

@Test fun load_emitsSuccess() = runTest {
    val repo = FakeUserRepo().apply { user = User(1, "A") }
    val vm = UserVm(repo)

    vm.state.test {
        assertEquals(UiState.Loading, awaitItem())
        vm.load(1)
        assertEquals(UiState.Success(User(1, "A")), awaitItem())
    }
}
```

Использовать Turbine для StateFlow/SharedFlow.

### Hilt в тестах

```kotlin
@HiltAndroidTest
class ProfileScreenTest {
    @get:Rule val hiltRule = HiltAndroidRule(this)
    @get:Rule val composeRule = createAndroidComposeRule<HiltTestActivity>()

    @BindValue @JvmField
    val fakeRepo: UserRepository = FakeUserRepo()

    @Test fun renders() {
        composeRule.setContent { ProfileScreen() }
        composeRule.onNodeWithText("Alice").assertExists()
    }
}
```

`@BindValue` подменяет binding в DI-графе.

### Compose UI tests

```kotlin
@get:Rule val composeRule = createComposeRule()

@Test fun counter_incrementsOnClick() {
    composeRule.setContent { Counter() }
    composeRule.onNodeWithText("0").assertExists()
    composeRule.onNodeWithContentDescription("Increment").performClick()
    composeRule.onNodeWithText("1").assertExists()
}
```

### Screenshot testing

**Paparazzi** (Square) — JVM, без эмулятора:

```kotlin
@get:Rule val paparazzi = Paparazzi(
    deviceConfig = DeviceConfig.PIXEL_5,
    theme = "AppTheme"
)

@Test fun profileCard() {
    paparazzi.snapshot {
        ProfileCard(user = User(1, "Alice"))
    }
}
```

При CI: сравнивает с golden image, fails если pixel diff. Ловит UI регрессии.

**Roborazzi** — то же через Robolectric, поддерживает Compose lifecycle.

### Integration tests (Robolectric)

```kotlin
@RunWith(RobolectricTestRunner::class)
class RoomIntegrationTest {
    private val db = Room.inMemoryDatabaseBuilder(...)

    @Test fun insertAndQuery() = runBlocking {
        db.userDao().insert(User(1, "A"))
        assertEquals(User(1, "A"), db.userDao().get(1))
    }
}
```

Реальная Room, без эмулятора (использует SQLite на JVM).

### End-to-end (Espresso / Maestro)

```yaml
# maestro flow:
- launchApp
- tapOn: "Login"
- inputText: "user@example.com"
- tapOn: "Submit"
- assertVisible: "Welcome"
```

Реальный device / emulator, реальная сеть (или MockWebServer).

### MockWebServer для network тестов

```kotlin
val server = MockWebServer().apply { start() }
server.enqueue(MockResponse().setBody("""{"id":1,"name":"A"}"""))

val api = Retrofit.Builder()
    .baseUrl(server.url("/"))
    .build()
    .create(UserApi::class.java)

val user = api.getUser(1)
assertEquals(User(1, "A"), user)
```

### Тестируемая архитектура

Архитектура напрямую влияет на тестируемость:

- **DI** — подмена реализаций.
- **Interfaces** — fake вместо impl.
- **Pure functions** в use cases — легко unit-тестить.
- **Inject Dispatcher** — TestDispatcher вместо `Dispatchers.IO`.
- **Stateless Composable** — preview-friendly, screenshot-friendly.
- **State hoisting** — тест верхнего state, child тривиален.

### Repository тестирование

```kotlin
@Test fun offlineFirst_returnsCacheIfNetworkFails() = runTest {
    val api = FakeUserApi(error = IOException())
    val dao = FakeUserDao(cached = listOf(UserEntity(1, "Cached")))
    val repo = UserRepositoryImpl(api, dao)

    val users = repo.observeUsers().first()
    assertEquals(listOf(User(1, "Cached")), users)
}
```

### Coverage цели (приблизительно)

- Domain (use cases, models) — 90%+.
- Data (repositories) — 80%+.
- ViewModels — 70%+.
- Composables / UI — screenshot + UI smoke.

100% coverage — не цель; coverage без проверок поведения — бессмысленно.

### CI integration

- Unit tests на каждом PR.
- Screenshot tests на каждом PR (failure → diff в комментарии).
- Integration tests — на main / nightly.
- E2E — nightly / pre-release.

## Подводные камни

- **Mock everything** → тест проверяет mock setup, а не реальное поведение.
- **Один большой test class** на 1000 строк → unmaintainable.
- **Flaky тесты** (зависят от timing) → обычно проблема с TestDispatcher / реальным delay.
- **UI тесты на эмуляторе** в CI без xvfb — медленные/flaky.
- **Screenshot tests** без version control golden images — diff не работает.
- **Тесты не запускаются** в CI → "проходят локально, ломают main".
- **Mock framework over-use** в Kotlin: final classes требуют `mockk` (Mockito с inline mocking).

## Связанные темы

- [[../05-Concurrency/q-14-coroutine-tests]]
- [[q-12-app-architecture-guide]]
- [[../12-Testing/q-test-pyramid]]

## Follow-up

- Чем fake лучше mock?
- Что такое test pyramid и почему?
- Зачем screenshot testing?
