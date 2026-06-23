---
type: question
category: architecture
difficulty: senior
tags: [clean-architecture, layers, usecase]
created: 2026-06-16
last_reviewed: 2026-06-16
times_failed: 0
times_passed: 1
status: learning
---

# Clean Architecture: слои, dependency rule, UseCase, Repository

## Краткий ответ (TL;DR)

Clean Architecture (Robert Martin) делит код на концентрические слои: **Entities (domain models)** → **Use Cases (business logic)** → **Interface Adapters (ViewModel, Repository impl, mappers)** → **Frameworks & Drivers (Android, Retrofit, Room)**. **Dependency rule**: зависимости направлены **внутрь** (внешние знают о внутренних, не наоборот). На Android применяется как 3 слоя: `data`, `domain`, `presentation`.

## Развёрнутый ответ

### Слои

```
┌─────────────────────────────────────┐
│  presentation (UI, ViewModel)       │  ─┐
│  ┌───────────────────────────────┐  │   │ depends on
│  │  domain (Entities, UseCase,   │  │   ▼
│  │           Repository iface)   │  │   ┌─────┐
│  │  ┌─────────────────────────┐  │  │   │     │
│  │  │  Entities (pure Kotlin) │  │  │   │ inward
│  │  └─────────────────────────┘  │  │   │     │
│  └───────────────────────────────┘  │   └─────┘
│  data (Retrofit, Room, Repository impl)
└─────────────────────────────────────┘
```

### Domain layer

Pure Kotlin модуль, **без Android-зависимостей**:

```kotlin
// domain/model/User.kt
data class User(val id: Long, val name: String, val email: String)

// domain/repository/UserRepository.kt
interface UserRepository {
    suspend fun getUser(id: Long): User
    fun observeUser(id: Long): Flow<User>
}

// domain/usecase/GetUserUseCase.kt
class GetUserUseCase(private val repo: UserRepository) {
    suspend operator fun invoke(id: Long): User = repo.getUser(id)
}
```

`operator fun invoke` → use case вызывается как функция: `getUser(id)`.

### Data layer

```kotlin
// data/remote/UserApi.kt
interface UserApi {
    @GET("users/{id}")
    suspend fun fetchUser(@Path("id") id: Long): UserDto
}

// data/local/UserEntity.kt (Room)
@Entity data class UserEntity(@PrimaryKey val id: Long, val name: String, val email: String)

// data/repository/UserRepositoryImpl.kt
class UserRepositoryImpl(
    private val api: UserApi,
    private val dao: UserDao
) : UserRepository {
    override suspend fun getUser(id: Long): User {
        val cached = dao.get(id)
        if (cached != null) return cached.toDomain()
        val dto = api.fetchUser(id)
        dao.insert(dto.toEntity())
        return dto.toDomain()
    }
    override fun observeUser(id: Long): Flow<User> =
        dao.observe(id).map { it.toDomain() }
}

// data/mapper/UserMappers.kt
fun UserDto.toDomain() = User(id, name, email)
fun UserEntity.toDomain() = User(id, name, email)
```

### Presentation layer

```kotlin
class UserVm(private val getUser: GetUserUseCase) : ViewModel() {
    private val _state = MutableStateFlow<UiState>(UiState.Loading)
    val state = _state.asStateFlow()

    fun load(id: Long) = viewModelScope.launch {
        _state.value = UiState.Loading
        runCatching { getUser(id) }
            .onSuccess { _state.value = UiState.Data(it.toUi()) }
            .onFailure { _state.value = UiState.Error(it.message ?: "") }
    }
}

// UI model
data class UserUi(val name: String, val displayEmail: String)
fun User.toUi() = UserUi(name, "✉ $email")
```

### Dependency rule в действии

- `domain` не знает о Retrofit/Room/Android.
- `data` знает о `domain` (имплементит интерфейсы), не знает о `presentation`.
- `presentation` знает о `domain`, не знает о `data` (только через DI).

### Зачем UseCase?

Зависит от сложности проекта:

- **Простой**: ViewModel напрямую вызывает Repository. UseCase = просто proxy.
- **Сложный**: UseCase инкапсулирует business logic (composition нескольких repo, валидация, side effects).

```kotlin
class PlaceOrderUseCase(
    private val cartRepo: CartRepository,
    private val orderRepo: OrderRepository,
    private val analytics: Analytics
) {
    suspend operator fun invoke(): Result<Order> {
        val cart = cartRepo.get()
        if (cart.isEmpty()) return Result.failure(EmptyCartException())
        val order = orderRepo.place(cart)
        analytics.log("order_placed", order.id)
        cartRepo.clear()
        return Result.success(order)
    }
}
```

### Аргументы за/против

**За**:
- Domain testable без Android (JVM-tests, быстрые).
- Можно поменять Room → SQLDelight, Retrofit → Ktor, не трогая domain/presentation.
- Чёткое разделение обязанностей.

**Против**:
- Boilerplate: 3 модели одной сущности (Dto, Entity, Domain, UI) + мапперы.
- Для маленьких приложений — overhead.
- UseCase часто = 1 строка → синтаксический шум.

Прагматичный подход: **слои есть всегда** (data/domain/presentation), **UseCase — по необходимости**.

### Модули vs пакеты

Clean можно реализовать просто пакетами:

```
com.x.app/
  data/
  domain/
  presentation/
```

Или Gradle модулями (см. [[q-08-modularization]]) — компилятор enforce'ит границы.

## Подводные камни

- Утечка Android-типов в domain (`Context`, `Uri`) → теряется суть. Использовать `String`/own wrapper.
- Маппинг Dto ↔ Domain ↔ UI на каждом запросе — может быть perf hit для больших списков. Профилируй.
- UseCase возвращает `LiveData`/`Flow` Android типа — норм для `Flow` (он kotlinx, не Android), `LiveData` — лучше не в domain.
- Repository метод возвращает `Result<T>` vs throws — выбирай один стиль и придерживайся.
- Чрезмерные UseCase для CRUD — `GetUserUseCase`, `SaveUserUseCase`, `DeleteUserUseCase` без логики — лучше прямой вызов Repo из VM.

## Связанные темы

- [[q-02-viewmodel]]
- [[q-08-modularization]]
- [[q-09-repository-pattern]]

## Follow-up

- Что такое dependency rule в Clean?
- Почему domain — pure Kotlin модуль без Android-зависимостей?
- Когда UseCase оправдан, а когда излишен?
