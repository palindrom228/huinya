---
type: question
category: android-core
difficulty: middle
tags: [resources, configuration, density, locale]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Ресурсы и Configuration: density, locale, night mode, размеры. Как Android выбирает ресурс?

## Краткий ответ (TL;DR)

Android выбирает ресурс по «качификаторам» в имени папки (`drawable-xxhdpi-v24`, `values-ru`, `layout-sw600dp-land`). Алгоритм отбора: фильтрация по неподходящим, ранжирование по приоритету (locale > screen size > density > version > ...). При изменении `Configuration` система может пересоздать Activity, ViewModel переживает.

## Развёрнутый ответ

### Структура ресурсов

```
res/
  values/strings.xml          ← default
  values-ru/strings.xml       ← русская локаль
  values-night/colors.xml     ← dark theme
  layout/main.xml
  layout-sw600dp/main.xml     ← tablet
  layout-land/main.xml        ← landscape
  drawable-mdpi/ic.png
  drawable-xhdpi/ic.png
  drawable-anydpi-v24/ic.xml  ← VectorDrawable от API 24
```

### Квалификаторы (приоритет сверху вниз)

1. MCC + MNC
2. Language + region (`values-ru-rRU`, BCP47 `values-b+zh+Hant+TW`)
3. Layout direction (`ldrtl` / `ldltr`)
4. Smallest width (`sw600dp`)
5. Available width / height (`w820dp` / `h720dp`)
6. Screen size (`small/normal/large/xlarge`) — устаревшее
7. Screen orientation (`port` / `land`)
8. UI mode (`car`, `desk`, `tv`, `watch`, `vrheadset`)
9. Night mode (`night` / `notnight`)
10. Density (`ldpi/mdpi/hdpi/xhdpi/xxhdpi/xxxhdpi/nodpi/anydpi/tvdpi/<NNN>dpi`)
11. Touch type / keyboard / navigation
12. Platform version (`v24`)

### Алгоритм выбора

1. Убирает все папки с **противоречащими** квалификаторами.
2. Из оставшихся выбирает с **самым высоким приоритетом** квалификатором, который различает их.

Пример: устройство xxhdpi, ru, ночь. Есть `drawable-xhdpi`, `drawable-xxhdpi-night`, `drawable-xxhdpi`.
- `drawable-xhdpi` подходит (downscaled), но `xxhdpi` лучше.
- `night` приоритетнее density.
- Выбор → `drawable-xxhdpi-night`.

### Densities

| Bucket | dpi | scale |
|--------|-----|-------|
| ldpi | 120 | 0.75x |
| mdpi | 160 | 1x (baseline) |
| hdpi | 240 | 1.5x |
| xhdpi | 320 | 2x |
| xxhdpi | 480 | 3x |
| xxxhdpi | 640 | 4x |

`dp` (density-independent pixel) → `px = dp * (dpi / 160)`.

Vector drawables (`anydpi-v24`) убирают необходимость в multiple-density bitmap'ах.

### Night mode

```kotlin
AppCompatDelegate.setDefaultNightMode(MODE_NIGHT_FOLLOW_SYSTEM)
```

Влияет на Configuration → ресурсы из `values-night/` подгружаются. Activity пересоздаётся, если в манифесте не указан `uiMode` в `configChanges`.

### Locale

```kotlin
// In-app language switching (AppCompat 1.6+)
AppCompatDelegate.setApplicationLocales(LocaleListCompat.forLanguageTags("ru-RU"))
```

Для библиотек / DI — `androidx.core.os.LocaleListCompat.getDefault()`.

### Configuration change

При смене locale, theme, orientation, density (multi-window resize), font scale → `Configuration` меняется. По умолчанию Activity пересоздаётся. Можно перехватить:

```xml
<activity android:configChanges="orientation|screenSize|uiMode|locale|layoutDirection|fontScale" />
```

Вызовется `onConfigurationChanged(newConfig)` без пересоздания. **Не рекомендуется** широко — много ловушек с ресурсами.

### Compose

Compose автоматически реагирует на configuration через `LocalConfiguration`, `MaterialTheme`, `isSystemInDarkTheme()` без пересоздания Activity (если используется Material3 + правильные тематические токены).

### Per-app language (Android 13+)

Системные настройки → язык на приложение. AppCompat 1.6+ предоставляет API:

```xml
<application android:localeConfig="@xml/locales_config" />
```

```xml
<!-- res/xml/locales_config.xml -->
<locale-config>
    <locale android:name="en" />
    <locale android:name="ru" />
</locale-config>
```

## Подводные камни

- `dp` vs `sp`: `sp` уважает font scale (для текста), `dp` — нет.
- `Resources.getString(R.string.x)` не пересоздаётся при смене locale в Activity, использующей старый Context (например, Application).
- `values-night` vs `values-v29-night` — нюансы для force dark mode.
- Vector drawable с тяжёлой path → драг на UI (lots of fill rule); profile рендеринг.
- Не все библиотеки уважают per-app language — нужно проверять inflate в правильном Context.

## Связанные темы

- [[q-01-activity-lifecycle]]
- [[../09-Compose/q-theming]]

## Follow-up

- Что произойдёт с Activity при изменении font scale?
- Чем `dp` отличается от `sp`?
- Как Android выбирает между `drawable-xhdpi` и `drawable-xxhdpi-night` для xxhdpi night-устройства?
