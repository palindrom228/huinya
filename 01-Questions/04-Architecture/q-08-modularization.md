---
type: question
category: architecture
difficulty: senior
tags: [modularization, gradle, multi-module]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Многомодульность: feature / core / data; benefits, anti-patterns

## Краткий ответ (TL;DR)

Делим проект на Gradle-модули: `:app` (entrypoint), `:core:*` (общая инфра), `:feature:*` (фичи), `:data` / `:domain` (слои). Выгоды: **параллельная компиляция**, **incremental build**, enforced boundaries (нельзя случайно импортнуть UI в data), reuse, dynamic features. Anti-patterns: god-module, циклические зависимости, утечка implementation через `api`.

## Развёрнутый ответ

### Типовая структура

```
:app                          # entrypoint, навигация, DI graph
:core
  :core-common                # utilities, base classes
  :core-ui                    # theme, design system
  :core-network               # Retrofit setup, interceptors
  :core-database              # Room setup, базовые DAO
:domain                       # pure Kotlin, models + use cases
:data                         # repositories, mappers
:feature
  :feature-home
    :feature-home-api         # public контракт (интерфейсы, модели для navigation)
    :feature-home-impl        # реализация (UI, VM)
  :feature-profile
    :feature-profile-api
    :feature-profile-impl
```

`:app` зависит от `:feature-*-impl`. `:feature` зависит от `:feature-*-api` (для cross-feature navigation), `:domain`, `:core-*`.

### Benefits

1. **Build performance**: Gradle параллелит модули, incremental: изменил `:feature-home`, остальные не пересобираются.
2. **Enforced boundaries**: feature не может импортнуть internals другого feature (только через api).
3. **Reusability**: `:core-network` переиспользуется во всех проектах команды.
4. **Dynamic Feature Modules**: загружаются on-demand через Play Feature Delivery.
5. **Team scaling**: команды владеют своими feature-модулями.

### implementation vs api

```kotlin
// :feature-home-impl/build.gradle.kts
dependencies {
    implementation(project(":core-network"))   // не пробрасывается дальше
    api(project(":domain"))                    // консьюмеры видят domain
}
```

- `implementation` — только этот модуль использует. Изменение public API не требует пересборки consumer'ов.
- `api` — пробрасывается транзитивно. Используй когда тип реально на границе.

Правило: **по умолчанию `implementation`**, `api` только когда необходимо.

### Convention plugins (Gradle)

`buildSrc` или `build-logic/` с pre-настроенными плагинами:

```kotlin
// build-logic/convention/AndroidLibraryConventionPlugin.kt
class AndroidLibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply("com.android.library")
            pluginManager.apply("org.jetbrains.kotlin.android")

            extensions.configure<LibraryExtension> {
                compileSdk = 34
                defaultConfig.minSdk = 24
            }
        }
    }
}
```

В feature-модуле:
```kotlin
plugins {
    id("myapp.android.library")
    id("myapp.android.library.compose")
}
```

Меняешь compileSdk в одном месте — обновляется во всех модулях.

### Cross-feature navigation

Проблема: `:feature-home` хочет открыть `:feature-profile`, но не может зависеть от него (циклы).

Решения:
1. **Common navigation module** (`:core-navigation`) с константами route'ов.
2. **API modules**: `:feature-profile-api` экспортирует `ProfileRoute.build(id)`, `:feature-home-impl` зависит от api.
3. **Deep link routing**: оба слушают глобальный intent flow.

```kotlin
// :feature-profile-api
object ProfileRoute {
    const val PATTERN = "profile/{id}"
    fun build(id: Long) = "profile/$id"
}

// :feature-home-impl
navController.navigate(ProfileRoute.build(42))
```

### Dynamic feature modules

```kotlin
// dynamic-feature module
plugins {
    id("com.android.dynamic-feature")
}

// app/build.gradle.kts
android {
    dynamicFeatures += setOf(":dynamic:premium")
}
```

Скачивается через `SplitInstallManager`. Подходит для редко используемых фич, локализаций по требованию.

### Anti-patterns

- **God-module `:common`** — всё подряд, все зависят от всего → теряются плюсы модуляризации.
- **Циклические зависимости** между feature-модулями → Gradle упадёт. Решение: общий api-модуль.
- **`api` везде** → транзитивные deps → пересборка лавиной.
- **Слишком мелкие модули** (`:feature-button`) → overhead Gradle конфигурации > выгоды.
- **`:domain` зависит от `:data`** → нарушение Clean. Domain должен быть pure.

### Build performance metrics

- `./gradlew --scan` → Gradle Build Scan показывает critical path.
- `gradle-doctor`, `gradle-profiler` для регрессий.
- KSP вместо KAPT (быстрее).
- `org.gradle.parallel=true`, `org.gradle.caching=true`, configuration cache.

## Подводные камни

- `:core-database` экспортирует `Room` через `api` → пересборка всех при `Room` версии bump.
- `:app` зависит от всех `:feature-*-impl` → изменение в любом feature пересобирает `:app` (это нормально, но `:app` лучше держать тонким).
- `implementation` в Android lib вместо `api` для UI-компонента → consumer не видит класс → compile error.
- Dynamic feature без `installLanguages` → локали не подтягиваются.
- Convention plugin без `buildSrc` или `build-logic/` includeBuild — не подхватится.

## Связанные темы

- [[q-03-clean-architecture]]
- [[q-06-hilt-dagger]]
- [[../10-Build-Tooling/q-gradle-basics]]

## Follow-up

- Почему лучше `implementation` чем `api` по умолчанию?
- Как организовать cross-feature навигацию без циклов?
- Что такое dynamic feature module?
