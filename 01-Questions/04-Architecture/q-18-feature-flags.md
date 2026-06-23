---
type: question
category: architecture
difficulty: middle
tags: [feature-flags, remote-config, ab-testing]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Feature flags архитектура: Remote Config, A/B, локальные overrides

## Краткий ответ (TL;DR)

Feature flag — boolean/enum/JSON, переключающий поведение в runtime. Источники: Firebase Remote Config, Unleash, GrowthBook, ConfigCat, локальный DataStore (для debug). Архитектура: **интерфейс `FeatureFlags`** в core, реализации в data (Firebase / fake / dev), доступ через DI. UI читает через VM → не зависит от провайдера. A/B testing — feature flag с распределением (50/50). Защита от багов в новой фиче — kill switch.

## Развёрнутый ответ

### Интерфейс

```kotlin
// :core-feature-flags
interface FeatureFlags {
    fun isEnabled(flag: Flag): Boolean
    fun <T> get(flag: Flag, default: T): T
    fun observe(flag: Flag): Flow<Boolean>
}

enum class Flag(val key: String, val default: Boolean = false) {
    NEW_CHECKOUT("new_checkout_enabled"),
    DARK_MODE_AUTO("dark_mode_auto", default = true),
    SEARCH_V2("search_v2_enabled")
}
```

### Firebase Remote Config реализация

```kotlin
class FirebaseFeatureFlags @Inject constructor(
    private val config: FirebaseRemoteConfig
) : FeatureFlags {

    init {
        config.setConfigSettingsAsync(
            remoteConfigSettings { minimumFetchIntervalInSeconds = 3600 }
        )
        config.fetchAndActivate()
    }

    override fun isEnabled(flag: Flag): Boolean =
        config.getBoolean(flag.key)
}
```

### Локальная override для debug

```kotlin
class DebugFeatureFlags @Inject constructor(
    private val remote: FirebaseFeatureFlags,
    private val dataStore: DataStore<Preferences>
) : FeatureFlags {
    override fun isEnabled(flag: Flag): Boolean = runBlocking {
        dataStore.data.first()[booleanPreferencesKey(flag.key)]
            ?: remote.isEnabled(flag)
    }
}
```

В debug билде разработчик может в Settings включать/выключать любой флаг локально.

### Использование в коде

```kotlin
class CheckoutVm @Inject constructor(
    private val flags: FeatureFlags,
    ...
) : ViewModel() {
    fun navigate() {
        if (flags.isEnabled(Flag.NEW_CHECKOUT)) {
            // новый экран
        } else {
            // старый
        }
    }
}
```

### Observable flags

```kotlin
class FeatureFlagsImpl(...) : FeatureFlags {
    override fun observe(flag: Flag): Flow<Boolean> =
        flowOf(isEnabled(flag))
            .onStart { fetchIfStale() }
}

@Composable
fun Screen() {
    val newUiEnabled by vm.observe(Flag.NEW_UI).collectAsStateWithLifecycle(false)
    if (newUiEnabled) NewLayout() else OldLayout()
}
```

### A/B testing

```kotlin
enum class Variant { A, B, CONTROL }

interface AbTesting {
    fun variant(experiment: String, userId: String): Variant
}

// Hash-based assignment:
class LocalAbTesting : AbTesting {
    override fun variant(experiment: String, userId: String): Variant {
        val hash = "$experiment:$userId".hashCode().absoluteValue
        return when (hash % 100) {
            in 0..49 -> Variant.A
            in 50..98 -> Variant.B
            else -> Variant.CONTROL
        }
    }
}
```

### Kill switch

```kotlin
if (flags.isEnabled(Flag.PAYMENT_DISABLED)) {
    return UiState.Error("Платежи временно недоступны")
}
```

Возможность мгновенно отключить багованную фичу через server-side flag без релиза.

### Архитектурные правила

- **`FeatureFlags` interface в core**, реализации в `:data`.
- UI / VM не знают какой провайдер.
- Default value у каждого флага (без сети должен работать).
- Cache локально (DataStore) — не запрашивать на каждый чек.
- Версионирование флагов (минимальная версия app, иначе игнорировать новый флаг).

### Cleanup флагов

Флаги — **temporary**. После полного rollout удалить:
- Сам ключ из remote config.
- `Flag.X` из enum.
- Все if/else проверки.

Иначе через год — code rot из десятков мёртвых флагов.

```kotlin
// Lint правило / unit test:
@Test fun noStaleFlags() {
    val flagAge = flags.allFlags().map { it.key to it.createdAt }
    flagAge.forEach { (key, created) ->
        assertTrue("Flag '$key' is older than 6 months — clean up",
            ChronoUnit.MONTHS.between(created, Instant.now()) < 6)
    }
}
```

### Multi-module флаги

```
:core-feature-flags          // interface + enum
:data-feature-flags-firebase // Firebase реализация
:data-feature-flags-fake     // for tests
:app                          // DI bind правильную impl per buildType
```

### Compose recomposition

```kotlin
@Composable
fun Screen() {
    val flag by flagsState   // StateFlow
    // recomposes когда flag меняется
}
```

Если флаг меняется в runtime (rotation, user toggles in dev panel) — UI обновится автоматически.

### Тестирование

```kotlin
class FakeFeatureFlags(private val map: Map<Flag, Boolean> = emptyMap()) : FeatureFlags {
    override fun isEnabled(flag: Flag) = map[flag] ?: flag.default
}

@Test fun newCheckoutFlow() {
    val vm = CheckoutVm(FakeFeatureFlags(mapOf(Flag.NEW_CHECKOUT to true)))
    // ...
}
```

### Server-side vs client-side

- **Client-side флаги** — Firebase Remote Config, локальные.
- **Server-side флаги** — feature flag сервис проверяет на backend, mobile получает уже отфильтрованные данные. Безопаснее для критичной логики (биллинг).

### Безопасность

- Не хранить **секреты** во флагах.
- Не использовать флаги для авторизации (можно подменить через MitM).
- Чувствительная логика — на сервере.

## Подводные камни

- Флаг без default value → сетевая ошибка ломает app.
- `isEnabled()` в hot loop → каждый чек запрос → лагает. Cache + observe.
- `runBlocking` для синхронного API → блокирует Main. Лучше pre-fetch.
- Mock VM без подмены flags → тест зависит от Firebase.
- Зомби-флаги (мёртвый код, забытые проверки) → cleanup discipline.
- Флаг для **feature**, который уже в production год — обнулить и удалить.

## Связанные темы

- [[q-08-modularization]]
- [[q-06-hilt-dagger]]
- [[../09-Build-Tooling/q-build-variants]]

## Follow-up

- Как организовать feature flags в multi-module?
- Что такое kill switch?
- Зачем cleanup старых флагов?
