---
type: question
category: architecture
difficulty: middle
tags: [di, koin, service-locator]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Koin, Service Locator vs DI, когда что выбирать

## Краткий ответ (TL;DR)

**Service Locator** — глобальный реестр, откуда код достаёт зависимости (`Locator.get<Repo>()`). Просто, но трудно тестировать (глобальное состояние). **DI** — зависимости передаются извне (конструктор, метод). **Koin** — DSL-based DI без annotation processing; работает через runtime resolution, по сути близок к Service Locator под капотом, но с DI-стилем API. Hilt лучше для больших Android-проектов; Koin для быстрых стартов / KMP.

## Развёрнутый ответ

### Service Locator anti-pattern

```kotlin
object Locator {
    val repo: UserRepository by lazy { UserRepositoryImpl(...) }
}

class UserVm : ViewModel() {
    private val repo = Locator.repo   // прячет зависимость
    fun load() = repo.fetch()
}
```

Проблемы:
- Зависимости не видны в сигнатуре класса.
- Тестировать → нужно подменять `Locator.repo` глобально.
- Coupling к Locator.

### Dependency Injection

```kotlin
class UserVm(private val repo: UserRepository) : ViewModel()   // зависимости явны
```

Кто создаёт `UserVm` (DI-framework, factory, тесты) — передаёт fake/mock.

### Koin

```kotlin
val appModule = module {
    single { OkHttpClient() }
    single { Retrofit.Builder().client(get()).baseUrl(URL).build() }
    single<UserApi> { get<Retrofit>().create() }
    single<UserRepository> { UserRepositoryImpl(get(), get()) }
    viewModel { UserVm(get()) }
}

class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidContext(this@MyApp)
            modules(appModule)
        }
    }
}

// Activity:
class MainActivity : AppCompatActivity() {
    private val vm: UserVm by viewModel()
}

// Compose:
@Composable
fun MyScreen(vm: UserVm = koinViewModel())
```

Scope keywords:
- `single` — singleton.
- `factory` — новый instance каждый раз.
- `scoped { ... }` — внутри `koinScope`.
- `viewModel` — связан с ViewModelStore.

### Koin vs Hilt

| | Koin | Hilt |
|--|------|------|
| Compile-time check | **нет** (crash в runtime если biding потерян) | да |
| Скорость старта | медленнее (build graph runtime) | быстрее (готовый код) |
| Boilerplate | минимум, DSL | annotations |
| KMP support | **да** | нет (Android only) |
| Learning | очень быстрый | средний |
| Подходит для | small/medium, KMP, прототипы | средние/большие Android |

Koin крутится на reflection + service locator под капотом → потенциальные runtime ошибки. На большом графе перформанс хуже.

### Manual DI (без фреймворка)

```kotlin
class AppContainer(context: Context) {
    private val okHttp = OkHttpClient()
    private val retrofit = Retrofit.Builder().client(okHttp).baseUrl(URL).build()
    val userApi: UserApi = retrofit.create()
    val userRepo: UserRepository = UserRepositoryImpl(userApi)
}

class MyApp : Application() {
    lateinit var container: AppContainer
    override fun onCreate() {
        super.onCreate()
        container = AppContainer(this)
    }
}
```

ViewModel получает через Factory. Для маленьких проектов / pet — нормально, без зависимостей на DI-framework.

### Когда что

- **Прототип / pet / KMP**: Koin или manual DI.
- **Small Android (<5 экранов)**: manual DI или Hilt — обе работают.
- **Production Android, multi-module**: Hilt (compile-time проверки спасают на большом графе).
- **Library**: чистый Dagger (без Hilt-зависимости на app component).

## Подводные камни

- Koin: добавили новый класс с зависимостями, забыли в module → **crash at runtime** (типичная ошибка).
- Service Locator в ViewModel — тесты требуют переинициализации singleton.
- Koin `viewModel { ... }` + Compose `koinViewModel()` без подключённого `koin-androidx-compose` — не работает.
- `single` в Koin = lazy singleton. Создаётся при первом get(). Eager — `single(createdAtStart = true)`.

## Связанные темы

- [[q-06-hilt-dagger]]
- [[../12-Testing/q-test-doubles]]

## Follow-up

- Чем Service Locator плох с точки зрения тестирования?
- Почему Koin не имеет compile-time check?
- Когда manual DI лучше любого фреймворка?
