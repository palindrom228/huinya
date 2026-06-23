---
type: question
category: architecture
difficulty: middle
tags: [architecture, google-guide, recommended]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# App Architecture Guidelines (Google): рекомендации, слои, типы данных

## Краткий ответ (TL;DR)

Google [App Architecture Guide](https://developer.android.com/topic/architecture) рекомендует **3 слоя**: **UI layer** (Activity/Fragment/Compose + ViewModel + UiState), **Domain layer** (опциональный — UseCase, бизнес-логика), **Data layer** (Repository + DataSource). Принципы: **separation of concerns**, **drive UI from data models** (immutable + observable), **single source of truth**, **unidirectional data flow**. State через `StateFlow`/`Flow`, events через `Channel`.

## Развёрнутый ответ

### Слои

```
┌──────────────────────────────────────┐
│  UI layer                            │
│  - Activity/Fragment / Composable    │
│  - ViewModel  (UiState, events)      │
└──────────────────┬───────────────────┘
                   │ depends on
┌──────────────────▼───────────────────┐
│  Domain layer (optional)             │
│  - UseCase / Interactor              │
└──────────────────┬───────────────────┘
                   │
┌──────────────────▼───────────────────┐
│  Data layer                          │
│  - Repository (interface + impl)     │
│  - DataSource (local, remote)        │
└──────────────────────────────────────┘
```

### UI layer

- **UI elements** (Composable/View) — рисуют state.
- **State holder** (ViewModel или plain class) — содержит `UiState`, обрабатывает logic.

```kotlin
data class NewsUiState(
    val items: List<NewsItem> = emptyList(),
    val isLoading: Boolean = false,
    val errorMessage: String? = null
)

class NewsVm @Inject constructor(
    private val repo: NewsRepository
) : ViewModel() {
    val uiState: StateFlow<NewsUiState> = repo.observeNews()
        .map { items -> NewsUiState(items = items) }
        .catch { emit(NewsUiState(errorMessage = it.message)) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = NewsUiState(isLoading = true)
        )
}
```

### Domain layer (optional)

Use case — переиспользуемая бизнес-логика, не привязанная к одному экрану.

```kotlin
class GetTopNewsUseCase @Inject constructor(
    private val newsRepo: NewsRepository,
    private val userRepo: UserRepository
) {
    operator fun invoke(): Flow<List<NewsItem>> =
        combine(newsRepo.observeNews(), userRepo.observeBookmarks()) { news, bookmarks ->
            news.map { it.copy(isBookmarked = it.id in bookmarks) }
        }
}
```

Когда нужен:
- Логика повторяется в нескольких VM.
- Сложная агрегация data sources.
- Хочется тестировать без VM.

### Data layer

```kotlin
interface NewsRepository {
    fun observeNews(): Flow<List<NewsItem>>
    suspend fun refresh()
}

class OfflineFirstNewsRepository @Inject constructor(
    private val dao: NewsDao,
    private val api: NewsApi
) : NewsRepository {

    override fun observeNews(): Flow<List<NewsItem>> =
        dao.observeAll().map { it.map(NewsEntity::asDomain) }

    override suspend fun refresh() {
        val remote = api.fetchTop()
        dao.upsertAll(remote.map { it.asEntity() })
    }
}
```

DataSource классы (опционально):
- `RemoteDataSource` — обёртка над Retrofit.
- `LocalDataSource` — обёртка над DAO.

Использовать когда хочется ещё один уровень абстракции (например, in-memory cache).

### Типы UiState

- **All-in-one data class** с nullable полями:
  ```kotlin
  data class UiState(val items: List<X>, val isLoading: Boolean, val error: String?)
  ```
  Просто, удобно для частичных обновлений.

- **Sealed UiState**:
  ```kotlin
  sealed interface UiState {
      data object Loading : UiState
      data class Success(val items: List<X>) : UiState
      data class Error(val msg: String) : UiState
  }
  ```
  Чётко описывает states, exhaustive when, но трудно для частичных обновлений (loading + items одновременно).

Google guide рекомендует **первый** подход для большинства случаев.

### Жизненный цикл наблюдения

`stateIn(WhileSubscribed(5_000))` — пайплайн активен пока есть подписчик + 5 сек после. Защищает от пересоздания при rotation, отменяет при долгом фоне.

В Compose:
```kotlin
val state by vm.uiState.collectAsStateWithLifecycle()
```

`collectAsStateWithLifecycle` — учитывает lifecycle (не collect'ит в background).

### Threading

- Suspend функции — **main-safe** (vivo, корректно сами переключают dispatcher через `withContext`).
- DataSource внутри использует `withContext(io)`.
- ViewModel не указывает dispatcher (полагается на main-safe API).

### Где модели делать immutable

Все: `data class` + `val`. Это базовое требование UDF.

### Naming conventions (Google)

- `*UiState` — UI state.
- `*ViewModel` — view model.
- `*Repository` — repository.
- `*DataSource` — низкоуровневый.
- `*UseCase` — use case (или `*Interactor`).

## Подводные камни

- VM выполняет I/O без `withContext` — главный поток заблокирован. Хотя suspend repo должна быть main-safe, защитный `withContext(io)` в repo — хорошая практика.
- `stateIn(Eagerly)` для тяжёлого Flow — даже без UI работает, тратит ресурсы.
- Domain layer как "ещё одна папка" без логики — boilerplate без пользы. Опциональный — значит опциональный.
- UI напрямую читает Repository (минуя VM) → теряется единая точка трансформации, тестируемость падает.
- Использование `LiveData` в data layer (Google guide рекомендует `Flow`).

## Связанные темы

- [[q-03-clean-architecture]]
- [[q-04-udf]]
- [[q-09-repository-pattern]]

## Follow-up

- Какие три слоя в Google App Architecture Guide?
- Когда нужен Domain layer (UseCase)?
- Зачем `stateIn(WhileSubscribed(5_000))`?
