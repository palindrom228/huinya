---
type: question
category: android-core
difficulty: senior
tags: [startup, performance, baseline-profiles]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# App startup: cold/warm/hot, Application.onCreate, App Startup библиотека, Baseline Profiles

## Краткий ответ (TL;DR)

**Cold** — процесс создаётся с нуля (Application + Activity). **Warm** — процесс жив, Activity создаётся. **Hot** — Activity жива, идёт в foreground. Оптимизация cold: минимизировать `Application.onCreate`, lazy init через **App Startup** или Hilt-EntryPoint, ускорить first frame через **Baseline Profiles**, профилировать с **Macrobenchmark**.

## Развёрнутый ответ

### Типы старта

| Тип | Что | Длительность ориентир |
|-----|-----|----------------------|
| Cold | Нет процесса, fork zygote → Application → Activity → first frame | 500ms – 3s+ |
| Warm | Процесс жив (кэш), Activity заново | ~200ms |
| Hot | Activity тоже жива, просто `onResume` | ~50ms |

Цели Google: cold < 5s, warm < 2s, hot < 1.5s. Хороший продукт — cold < 1s на топ-устройствах.

### Что происходит при cold start

1. **Zygote fork** — Linux fork процесса с предзагруженным runtime.
2. **Application.attachBaseContext** — context установлен.
3. **ContentProvider.onCreate** для всех провайдеров в манифесте (включая FirebaseInitProvider, WorkManagerInitializer, AndroidX-Startup).
4. **Application.onCreate**.
5. **Activity.onCreate → onStart → onResume**.
6. **Window decor inflate** → measure/layout/draw.
7. **First frame** committed → reported в `reportFullyDrawn()`.

### Где обычно теряются миллисекунды

- Heavy SDK init в `Application.onCreate` (Firebase, Amplitude, Adjust, Sentry, Datadog).
- Lots of ContentProvider'ов от библиотек.
- DI graph build (Dagger) — сильно зависит от размера графа.
- DataStore/Room первый доступ — миграции, file open.
- Heavy layout inflate с deep view tree.
- Network call синхронно (никогда так не делать).
- Reflection-based libs.

### AndroidX Startup

Замена N ContentProvider'ов на один + dependency graph:

```kotlin
class TimberInitializer : Initializer<Unit> {
    override fun create(context: Context) {
        if (BuildConfig.DEBUG) Timber.plant(Timber.DebugTree())
    }
    override fun dependencies() = emptyList<Class<out Initializer<*>>>()
}
```

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    tools:node="merge">
    <meta-data android:name="com.x.TimberInitializer"
               android:value="androidx.startup" />
</provider>
```

Можно **выключить eager init** и вызвать вручную:

```xml
<meta-data android:name="com.x.HeavyInitializer" tools:node="remove" />
```

```kotlin
AppInitializer.getInstance(context).initializeComponent(HeavyInitializer::class.java)
```

### Lazy init

```kotlin
val analytics by lazy { Analytics(context) }   // создастся при первом обращении
```

Hilt: `Provider<T>` или `Lazy<T>` для отложенной инстанциации.

### Splash Screen API (Android 12+)

```xml
<style name="Theme.App.Splash" parent="Theme.SplashScreen">
    <item name="windowSplashScreenBackground">@color/brand</item>
    <item name="windowSplashScreenAnimatedIcon">@drawable/ic_splash</item>
    <item name="postSplashScreenTheme">@style/Theme.App</item>
</style>
```

```kotlin
val splash = installSplashScreen()
splash.setKeepOnScreenCondition { viewModel.isLoading.value }
```

Не делать «фейковый splash» на отдельной Activity — это удваивает cold start.

### Baseline Profiles

Файл `baseline-prof.txt` в APK/AAB указывает ART, какие **классы и методы AOT-компилировать** при установке (без profile только JIT, что медленнее на cold start).

Генерация:

```kotlin
@BaselineProfileRule
class BaselineProfileGenerator {
    @get:Rule val rule = BaselineProfileRule()
    @Test fun generate() = rule.collect("com.x.app") {
        startActivityAndWait()
        // user journey: scroll, click...
    }
}
```

Результат — `app/src/main/baseline-prof.txt`. Польза на cold start — **30–40%** улучшения на cold start и scroll perf.

### Macrobenchmark

```kotlin
@RunWith(AndroidJUnit4::class)
class StartupBenchmark {
    @get:Rule val benchmarkRule = MacrobenchmarkRule()

    @Test fun startup() = benchmarkRule.measureRepeated(
        packageName = "com.x.app",
        metrics = listOf(StartupTimingMetric()),
        iterations = 10,
        startupMode = StartupMode.COLD
    ) {
        pressHome()
        startActivityAndWait()
    }
}
```

Измеряет TTID (time to initial display) и TTFD (time to fully drawn).

### reportFullyDrawn

```kotlin
override fun onResume() {
    super.onResume()
    // когда контент реально загружен и нарисован
    reportFullyDrawn()
}
```

Системой используется для метрик; Macrobenchmark берёт это значение в TTFD.

## Подводные камни

- `Application.onCreate` вызывается в **каждом процессе** → multi-process приложения инициализируют всё дважды.
- Firebase/Crashlytics ContentProvider — auto-init; чтобы отключить → `tools:node="remove"` + manual init.
- Baseline Profile без обновления при больших изменениях кода устаревает.
- Splash на отдельной Activity → лишний `onCreate → onDestroy` цикл.
- WorkManager `onCreate` тяжёлый, если используешь `Configuration.Provider` — он откладывает init до первого `getInstance`.

## Связанные темы

- [[q-12-content-provider]]
- [[../07-Performance-Memory/q-jank]]
- [[../06-Build/q-r8-shrink]]

## Follow-up

- Чем cold start отличается от warm? Что между ними «между»?
- Как baseline profile ускоряет первый запуск?
- Почему нельзя ставить heavy init в `Application.onCreate`?
