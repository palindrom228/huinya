---
type: question
category: architecture
difficulty: senior
tags: [di, hilt, dagger]
created: 2026-06-16
last_reviewed: 2026-06-16
times_failed: 0
times_passed: 1
status: learning
---

# DI: Hilt vs Dagger 2, scope-ы, multibinding, Assisted Inject

## Краткий ответ (TL;DR)

**Dagger 2** — compile-time DI через annotation processing (KSP/KAPT), генерит код. Очень мощный, но boilerplate. **Hilt** — Google-обёртка над Dagger с pre-defined компонентами для Android (Application, Activity, Fragment, ViewModel). Scope = время жизни instance (`@Singleton`, `@ActivityScoped`, `@ViewModelScoped`). **Multibinding** = `@IntoSet`/`@IntoMap` для коллекций (например, plugin система). **Assisted Inject** — для параметров, известных только в runtime.

## Развёрнутый ответ

### Hilt setup

```kotlin
@HiltAndroidApp
class MyApp : Application()

@AndroidEntryPoint
class MainActivity : ComponentActivity()

@HiltViewModel
class UserVm @Inject constructor(
    private val repo: UserRepository
) : ViewModel()
```

### Module

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides @Singleton
    fun provideRetrofit(okHttp: OkHttpClient): Retrofit =
        Retrofit.Builder().client(okHttp).baseUrl(BASE).build()

    @Provides @Singleton
    fun provideUserApi(retrofit: Retrofit): UserApi = retrofit.create()
}

@Module
@InstallIn(SingletonComponent::class)
abstract class RepoModule {
    @Binds @Singleton
    abstract fun bindUserRepo(impl: UserRepositoryImpl): UserRepository
}
```

`@Provides` для конструкции; `@Binds` (abstract) — для биндинга интерфейс → имплементация (генерит меньше кода).

### Hilt компоненты и scope

| Component | Scope | Lifetime |
|-----------|-------|----------|
| `SingletonComponent` | `@Singleton` | Application |
| `ActivityRetainedComponent` | `@ActivityRetainedScoped` | переживает rotation (как ViewModel) |
| `ViewModelComponent` | `@ViewModelScoped` | один ViewModel instance |
| `ActivityComponent` | `@ActivityScoped` | одна Activity (rebuilt on rotation) |
| `FragmentComponent` | `@FragmentScoped` | один Fragment |
| `ViewComponent` | `@ViewScoped` | один View |
| `ServiceComponent` | `@ServiceScoped` | один Service |

Без scope — каждый `@Inject` создаёт новый instance.

### Qualifiers

```kotlin
@Qualifier @Retention(BINARY) annotation class IoDispatcher
@Qualifier @Retention(BINARY) annotation class MainDispatcher

@Provides @IoDispatcher
fun provideIo() = Dispatchers.IO

@Provides @MainDispatcher
fun provideMain() = Dispatchers.Main

class Repo @Inject constructor(
    @IoDispatcher private val io: CoroutineDispatcher
)
```

Решает «два biding'а одного типа».

### Multibinding

```kotlin
@Module @InstallIn(SingletonComponent::class)
abstract class AnalyticsModule {
    @Binds @IntoSet
    abstract fun bindFirebase(impl: FirebaseAnalytics): AnalyticsTracker

    @Binds @IntoSet
    abstract fun bindAmplitude(impl: AmplitudeAnalytics): AnalyticsTracker
}

class Analytics @Inject constructor(
    private val trackers: Set<@JvmSuppressWildcards AnalyticsTracker>
) {
    fun log(event: String) = trackers.forEach { it.log(event) }
}
```

`@IntoMap`:

```kotlin
@Binds @IntoMap @StringKey("home")
abstract fun bindHome(impl: HomeScreen): Screen

inject: Map<String, @JvmSuppressWildcards Screen>
```

### Assisted Inject

Параметр известен в runtime, не из графа:

```kotlin
@HiltViewModel(assistedFactory = DetailVm.Factory::class)
class DetailVm @AssistedInject constructor(
    @Assisted val itemId: Long,
    private val repo: ItemRepository
) : ViewModel() {
    @AssistedFactory interface Factory {
        fun create(itemId: Long): DetailVm
    }
}

@Composable
fun DetailScreen(itemId: Long) {
    val vm = hiltViewModel<DetailVm, DetailVm.Factory> { factory ->
        factory.create(itemId)
    }
}
```

### EntryPoint

Получить зависимости в местах, не управляемых Hilt (Service, BroadcastReceiver, ContentProvider, Worker без HiltWorker, library code):

```kotlin
@EntryPoint @InstallIn(SingletonComponent::class)
interface MyEntryPoint {
    fun userRepo(): UserRepository
}

val ep = EntryPointAccessors.fromApplication(context, MyEntryPoint::class.java)
val repo = ep.userRepo()
```

### Hilt vs Dagger vs Koin

| | Hilt | Dagger | Koin |
|--|------|--------|------|
| Compile-time check | да | да | нет (runtime) |
| Performance | компилируется в код | то же | reflection-heavy → медленнее |
| Boilerplate | низкий (Android-frienly) | высокий | очень низкий |
| Android-aware | да | нет (свои Components) | через extensions |
| Learning curve | средний | высокий | низкий |

Hilt — стандарт для нового Android. Dagger — где Hilt не подходит (KMP, library). Koin — для маленьких / прототипов.

### Modularization + Hilt

Hilt автоматически собирает `@Module`-ы из всех модулей в граф. Для feature-модулей:

```kotlin
@Module @InstallIn(SingletonComponent::class)
object FeatureXModule {
    @Provides fun provideX(): FeatureXApi = ...
}
```

API публикуется через `:domain` или `:api` модуль, реализация в `:data`.

### KSP migration

Hilt с 2.48 поддерживает **KSP** (быстрее KAPT). В Gradle:

```kotlin
plugins {
    id("com.google.devtools.ksp")
    id("dagger.hilt.android.plugin")
}
dependencies {
    implementation("com.google.dagger:hilt-android:2.51")
    ksp("com.google.dagger:hilt-android-compiler:2.51")
}
```

## Подводные камни

- `@HiltViewModel` без `@AndroidEntryPoint` на host Activity → crash при создании VM.
- Scope несоответствие: `@Singleton` зависит от `@ActivityScoped` → compile error («may not reference bindings with different scopes»).
- `@Inject` на field в Activity без `@AndroidEntryPoint` → не работает.
- Multibinding без `@JvmSuppressWildcards` → wildcard mismatch (Kotlin vs Java generics).
- `@Provides` метод с suspend — не поддерживается. Обернуть в provider/lazy.
- Worker инджект через `@HiltWorker` + `@AssistedInject` (Worker имеет два assisted-параметра: context, params).

## Связанные темы

- [[q-02-viewmodel]]
- [[q-07-koin-service-locator]]
- [[q-08-modularization]]

## Follow-up

- Чем Hilt отличается от чистого Dagger?
- Зачем `@Qualifier`?
- Когда нужен `@AssistedInject`?
