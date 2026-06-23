---
type: question
category: architecture
difficulty: middle
tags: [repository, data-layer, mappers]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Repository pattern: маперы, DTO/Domain/UI модели, кеш

## Краткий ответ (TL;DR)

**Repository** — абстракция доступа к данным, скрывающая источник (network, DB, in-memory). Возвращает **domain models**, а не Retrofit DTO. **DTO** — wire-format (json mirror); **Entity** — таблица БД; **Domain** — бизнес-модель; **UI** — то, что рисуется. Мапперы конвертируют между слоями. Repository — SSoT для своих данных, часто реализует offline-first через DB+Flow.

## Развёрнутый ответ

### Интерфейс в domain

```kotlin
// :domain
interface UserRepository {
    fun observeUser(id: Long): Flow<User>
    suspend fun refresh(id: Long)
    suspend fun update(user: User)
}

data class User(val id: Long, val name: String, val email: String)
```

### Имплементация в data

```kotlin
// :data
class UserRepositoryImpl(
    private val api: UserApi,
    private val dao: UserDao,
    @IoDispatcher private val io: CoroutineDispatcher
) : UserRepository {

    override fun observeUser(id: Long): Flow<User> =
        dao.observe(id)
            .filterNotNull()
            .map { it.toDomain() }
            .flowOn(io)

    override suspend fun refresh(id: Long) = withContext(io) {
        val dto = api.fetchUser(id)
        dao.upsert(dto.toEntity())
    }

    override suspend fun update(user: User) = withContext(io) {
        api.updateUser(user.toDto())
        dao.upsert(user.toEntity())
    }
}
```

### Модели и мапперы

```kotlin
// data/remote
@Serializable
data class UserDto(
    @SerialName("id") val id: Long,
    @SerialName("full_name") val fullName: String,
    @SerialName("email_address") val email: String
)
fun UserDto.toDomain() = User(id, fullName, email)
fun UserDto.toEntity() = UserEntity(id, fullName, email)

// data/local
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: Long,
    val name: String,
    val email: String
)
fun UserEntity.toDomain() = User(id, name, email)

// domain → wire
fun User.toDto() = UserDto(id, name, email)
fun User.toEntity() = UserEntity(id, name, email)

// presentation
data class UserUi(val displayName: String, val maskedEmail: String)
fun User.toUi() = UserUi(
    displayName = name.uppercase(),
    maskedEmail = email.replaceBefore("@", "***")
)
```

### Почему отдельные модели

- **DTO**: завязана на формат API (snake_case, nested, опциональные поля). Меняется когда меняется backend.
- **Entity**: завязана на схему БД (indices, FK, type converters).
- **Domain**: бизнес-логика. Стабильна.
- **UI**: форматирование (currency, дата, маска).

Разделение → backend меняет field name → правим DTO + mapper, не трогая domain/UI.

### Offline-first

```kotlin
override fun observeProducts(): Flow<List<Product>> = flow {
    // 1. сразу эмитим из DB (мгновенный UI)
    emitAll(dao.observeAll().map { list -> list.map { it.toDomain() } })
}

suspend fun sync() {
    val remote = api.fetchAll()
    dao.upsertAll(remote.map { it.toEntity() })
    // Flow подписчики получат update автоматически
}
```

### Стратегии кеша

- **Cache-first**: вернуть из DB, фоном refresh.
- **Network-first**: попробовать сеть, fallback на DB.
- **Cache-only / network-only**: явные методы.
- **Stale-while-revalidate** (SWR): вернуть кеш, параллельно обновить.

### Multiple data sources

```kotlin
class ProductRepository(
    private val remote: ProductRemoteDataSource,
    private val local: ProductLocalDataSource,
    private val memory: InMemoryCache<Long, Product>
) {
    suspend fun get(id: Long): Product =
        memory.get(id) ?: local.get(id) ?: remote.fetch(id).also {
            local.save(it)
            memory.put(id, it)
        }
}
```

`DataSource` классы — приватные для repository, отделяют технические детали.

### Result vs Exception

```kotlin
// Option A: throws
suspend fun refresh(id: Long): Unit  // throws IOException

// Option B: Result wrapper
sealed interface Result<out T> {
    data class Success<T>(val value: T) : Result<T>
    data class Failure(val error: Throwable) : Result<Nothing>
}

suspend fun refresh(id: Long): Result<Unit>

// Option C: kotlin.Result (с осторожностью — generic limitations)
```

Выбор — стилевой; **один** стиль на проект.

### Pagination

```kotlin
fun pagedProducts(): Flow<PagingData<Product>> =
    Pager(PagingConfig(pageSize = 20)) {
        ProductPagingSource(api)
    }.flow
```

См. [[../06-Data-Networking/q-paging]].

## Подводные камни

- Repository возвращает DTO → presentation знает формат сети → coupling.
- Mapping в hot path (1000 элементов на каждый emit) → перф hit. Профилируй.
- Repository + race condition: два concurrent refresh → дубли в DB. Использовать `Mutex` или `flatMapLatest`.
- Кеш без TTL/invalidation → stale data навсегда.
- Mapper с throwing constructor (NPE на nullable field) → весь Flow ломается. Provide defaults / nullable domain field.

## Связанные темы

- [[q-03-clean-architecture]]
- [[q-04-udf]]
- [[../06-Data-Networking/q-room]]

## Follow-up

- Зачем отдельные DTO и Domain модели?
- Что такое offline-first и как реализуется через Room+Flow?
- Repository возвращает throws или Result — какой стиль выбрать?
