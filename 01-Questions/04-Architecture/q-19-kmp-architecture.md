---
type: question
category: architecture
difficulty: senior
tags: [kmp, multiplatform, sharing]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# KMP / Kotlin Multiplatform: что и как шарить, expect/actual

## Краткий ответ (TL;DR)

**Kotlin Multiplatform** — общий код (бизнес-логика, network, БД) для Android/iOS/Desktop/Web. **Shared core** обычно содержит: models, repositories, use cases, networking (Ktor), DB (SQLDelight), DI (Koin). UI — нативный (Compose Multiplatform — опционально). `expect/actual` для platform-specific (Settings, Date, Logger). На iOS — KMP компилируется в Objective-C / Swift-friendly framework. Архитектура: Clean / MVVM в shared, нативное UI поверх.

## Развёрнутый ответ

### Модульная структура KMP проекта

```
:shared
  commonMain/         // общий код
  androidMain/        // Android-specific
  iosMain/            // iOS-specific
  commonTest/

:androidApp           // Android-only, UI на Compose
:iosApp               // iOS, Swift, использует shared.framework
```

### commonMain содержит

- **Domain models** — `data class User(...)`.
- **Repositories** — interface + impl.
- **Use cases** — pure business logic.
- **Networking** — Ktor Client (multiplatform).
- **DB** — SQLDelight (multiplatform).
- **Serialization** — kotlinx.serialization.
- **Coroutines + Flow** — работают на всех таргетах.
- **DI** — Koin (multiplatform).

### expect / actual

```kotlin
// commonMain
expect class PlatformLogger() {
    fun log(message: String)
}

// androidMain
actual class PlatformLogger {
    actual fun log(message: String) = Log.d("KMP", message)
}

// iosMain
actual class PlatformLogger {
    actual fun log(message: String) = NSLog("KMP %@", message)
}
```

Используется для:
- Logger / Crash reporting.
- DataStore / NSUserDefaults.
- Date / NSDate.
- HttpClient engine (OkHttp vs Darwin).

### Ktor client

```kotlin
// commonMain
val client = HttpClient {
    install(ContentNegotiation) { json() }
    install(Logging)
}

// androidMain
HttpClient(OkHttp) { ... }

// iosMain
HttpClient(Darwin) { ... }
```

### SQLDelight

```sql
-- shared/sqldelight/db/UserQueries.sq
CREATE TABLE user (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL
);

selectAll:
SELECT * FROM user;
```

Генерирует type-safe Kotlin код, работает на Android (SQLite) и iOS (native SQLite).

### ViewModel в KMP

Android ViewModel — Android-only. В KMP используют:

**Вариант 1**: ViewModel в androidMain, iOS пишет свой StateObject:

```kotlin
// commonMain
class SharedPresenter(private val repo: Repo) {
    private val _state = MutableStateFlow(...)
    val state: StateFlow<UiState> = _state.asStateFlow()
}

// androidMain
class HomeVm(...) : ViewModel() {
    val presenter = SharedPresenter(...)
}

// iOS Swift
class HomeViewModel: ObservableObject {
    let presenter: SharedPresenter
}
```

**Вариант 2**: Decompose / MVIKotlin / Voyager — multiplatform альтернативы Android ViewModel.

**Вариант 3**: Google AndroidX ViewModel multiplatform (experimental) — `androidx.lifecycle:lifecycle-viewmodel` доступен в KMP.

### Flow для iOS

Корутиновый Flow на iOS не работает напрямую. Решения:
- **SKIE** (Touchlab) — генерирует Swift-friendly типы для Flow.
- **kmp-nativecoroutines** — async/await wrapper.
- Ручной wrapper c callback'ами.

```kotlin
// commonMain
fun observeUsers(): Flow<List<User>> = repo.observeUsers()

// iOS Swift (с SKIE):
for try await users in presenter.observeUsers() { ... }
```

### Dependency Injection

**Koin** — основной выбор для KMP:

```kotlin
// commonMain
val sharedModule = module {
    single { UserRepository(get(), get()) }
    factory { GetUsersUseCase(get()) }
}

// androidMain
val androidModule = module {
    single<HttpClient> { HttpClient(OkHttp) }
}

// init:
startKoin { modules(sharedModule, androidModule) }
```

Hilt в KMP не работает (Android-only).

### UI: Compose Multiplatform vs native

- **Native UI** (Compose Android + SwiftUI iOS) — лучший UX, больше работы.
- **Compose Multiplatform** (JetBrains) — один UI на оба. Растёт зрелость, но iOS pixel-perfect ещё не везде.

Для production приложения с серьёзным iOS UX — обычно native UI + shared business logic.

### Сериализация

```kotlin
@Serializable
data class User(val id: Long, val name: String)

// kotlinx.serialization работает на всех таргетах
```

### Архитектура слоёв в KMP

```
commonMain
  domain/           — models, use cases
  data/             — repository, network, db
  presentation/     — StateFlow, presenter (опционально)

androidMain
  ui/               — Composable + AndroidX ViewModel

iosApp (Swift)
  Views/            — SwiftUI
  ViewModels/       — ObservableObject обёртка над shared
```

### Тестирование

```kotlin
// commonTest
class UserRepoTest {
    @Test fun loadUser() = runTest {
        val repo = UserRepository(FakeApi(), FakeDao())
        assertEquals(expectedUser, repo.getUser(1))
    }
}
```

Тесты в `commonTest` запускаются на всех таргетах.

### Сложности KMP

- **iOS interop**: классы становятся ObjC-style (`KotlinFlow`, длинные имена). SKIE улучшает.
- **Hot reload**: на iOS медленнее (нужно пересобрать framework).
- **Build time**: gradle + cocoapods + xcode — медленнее чисто-Android.
- **Native dependencies**: библиотека с C/C++ — отдельная работа.
- **Memory model**: до Kotlin 1.7 — old memory model на iOS (frozen objects). Сейчас new memory model по умолчанию.

### Когда KMP оправдан

✅ Стабильная бизнес-логика, дублируется между Android и iOS.
✅ Маленькая команда хочет шарить repository / network / DB layer.
✅ Высокая частота изменений в бизнес-логике — KMP избавляет от sync двух команд.

❌ Команда не знает Kotlin (iOS dev).
❌ Простое приложение, бизнес-логики мало.
❌ Уже большой codebase в native — миграция дорогая.

### KMP vs альтернативы

- **Flutter / React Native** — общий и UI, и логика. Но native interop сложнее.
- **KMP** — общая логика, нативный UI. Меньше компромиссов по UX.

## Подводные камни

- Frozen objects на iOS (старая memory model) → InvalidMutabilityException.
- Coroutines Job/Flow в Swift без wrapper'ов — неудобно.
- Подключение native iOS зависимости через CocoaPods → flaky gradle sync.
- `expect` без `actual` в каком-то таргете → compile error только на этом таргете.
- Большой framework (200+MB) → app size impact на iOS. Strip / optimize.
- Hilt в shared → не скомпилируется на iOS.

## Связанные темы

- [[q-10-mvi-orbit-mvikotlin]]
- [[q-08-modularization]]
- [[../06-Data-Networking/q-ktor-vs-retrofit]]

## Follow-up

- Что шарить, что не шарить в KMP?
- Зачем `expect/actual` и где использовать?
- Как iOS работает с Kotlin Flow?
