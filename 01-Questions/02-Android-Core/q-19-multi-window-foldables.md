---
type: question
category: android-core
difficulty: senior
tags: [multi-window, foldables, configuration]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Multi-window, split-screen, foldables. Window size classes, posture

## Краткий ответ (TL;DR)

С Android 7+ multi-window — реальность: split-screen, free-form, picture-in-picture. С Android 10+ — multi-resume (несколько Activity в RESUMED). Foldables меняют размер экрана динамически. Адаптивный UI делать через **WindowSizeClass** (Compose/AppCompat) + **Jetpack WindowManager** для posture (folded, half-opened, table-top).

## Развёрнутый ответ

### Multi-window типы

- **Split-screen** — пользователь разделяет экран на две Activity.
- **Free-form** — оконный режим (планшеты, ChromeOS, Samsung DeX).
- **Picture-in-picture (PiP)** — маленькое окошко поверх (видео).
- **Multi-resume** — обе Activity в `RESUMED` (Android 10+). Раньше только верхняя была `RESUMED`, остальные `PAUSED`.

### resizeableActivity

```xml
<application android:resizeableActivity="true">
```

С `targetSdk >= 24` по умолчанию `true`. Если `false` — система запускает в **compatibility mode** (черные полосы при split-screen). Play Store / OEM штрафуют `false`.

### Lifecycle нюансы

При split-screen:
- Когда фокус НЕ у вашей Activity → `onPause` (но НЕ `onStop`).
- Когда фокус у вашей Activity → `onResume`.
- При resize окна → `onConfigurationChanged` (если перехвачен) или `onDestroy → onCreate`.

Слушать **focus**:

```kotlin
override fun onWindowFocusChanged(hasFocus: Boolean) {
    if (hasFocus) startCameraPreview() else pause()
}
```

Раньше использовали `onResume/onPause` — теперь неверно (вы в `onResume`, но без фокуса).

### Подписки на ресурсы

Запускать camera/sensors не в `onResume`, а при `hasWindowFocus`. Иначе две Activity делят камеру → конфликт.

### Picture-in-Picture

```xml
<activity android:supportsPictureInPicture="true"
          android:configChanges="screenSize|smallestScreenSize|screenLayout|orientation" />
```

```kotlin
enterPictureInPictureMode(
    PictureInPictureParams.Builder()
        .setAspectRatio(Rational(16, 9))
        .build()
)

override fun onPictureInPictureModeChanged(isInPip: Boolean, newConfig: Configuration) {
    if (isInPip) hideUiControls() else showUiControls()
}
```

### WindowSizeClass (адаптивный UI)

```kotlin
// Compose Material3
val windowSizeClass = calculateWindowSizeClass(activity)
when (windowSizeClass.widthSizeClass) {
    WindowWidthSizeClass.Compact -> SinglePane()
    WindowWidthSizeClass.Medium -> ListDetailLight()
    WindowWidthSizeClass.Expanded -> ListDetail()
}
```

Breakpoints:
- **Compact**: < 600dp (phone portrait).
- **Medium**: 600–840dp (tablet portrait, phone landscape, unfolded foldable portrait).
- **Expanded**: ≥ 840dp (tablet landscape, foldable unfolded landscape, desktop).

### Jetpack WindowManager (foldables)

```kotlin
WindowInfoTracker.getOrCreate(this).windowLayoutInfo(this)
    .collect { info ->
        val fold = info.displayFeatures.filterIsInstance<FoldingFeature>().firstOrNull()
        when (fold?.state) {
            FoldingFeature.State.HALF_OPENED -> tableTopUi(fold)
            FoldingFeature.State.FLAT -> normalUi()
            null -> normalUi()
        }
    }
```

`FoldingFeature` даёт **bounds** сгиба и **orientation** (вертикальный/горизонтальный) — можно избегать важных элементов под сгибом.

Postures:
- **Flat** — обычный планшет.
- **Half-opened vertical (book)** — две страницы.
- **Half-opened horizontal (tabletop)** — видео сверху, controls снизу.

### SlidingPaneLayout / TwoPaneLayout

Адаптивные layout'ы:
- `SlidingPaneLayout` (AppCompat) — на узких экранах детальная панель «выезжает», на широких видны обе.
- Compose `AdaptiveListDetailPane` (Material3-adaptive).

## Подводные камни

- `android:configChanges="orientation|screenSize"` чтобы перехватить resize → не получишь корректные ресурсы. Лучше пусть пересоздаётся + ViewModel.
- Запуск второй копии Activity в split-screen → если launchMode `singleTask` → пользователь не сможет открыть в обоих окнах.
- Camera/microphone в `onResume` без focus check → конфликт между окнами.
- PiP без `configChanges` → каждый resize пересоздаёт Activity → видео перезапускается.
- `WindowMetrics.getCurrentWindowMetrics()` (API 30+) — корректный размер окна, не экрана. Старый `Display.getMetrics` врёт в multi-window.

## Связанные темы

- [[q-01-activity-lifecycle]]
- [[q-14-resources-configuration]]
- [[../09-Compose/q-adaptive-layouts]]

## Follow-up

- Чем `onResume` отличается от `onWindowFocusChanged` в multi-window?
- Какие window size class breakpoints?
- Как обнаружить, что foldable находится в tabletop mode?
