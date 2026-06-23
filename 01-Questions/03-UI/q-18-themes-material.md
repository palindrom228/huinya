---
type: question
category: ui
difficulty: middle
tags: [theming, material3, dark-mode]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Темы, Material 3, dark mode, dynamic color

## Краткий ответ (TL;DR)

Material 3 (Material You) — текущий дизайн-системы Google. В Compose: `MaterialTheme(colorScheme, typography, shapes)`. Dark mode: `isSystemInDarkTheme()` + `darkColorScheme()`. **Dynamic color** (Android 12+): `dynamicLightColorScheme(context)` берёт цвета из обоев системы. Для XML — `Theme.Material3.DayNight.*` + `values-night`.

## Развёрнутый ответ

### Compose Material 3

```kotlin
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= 31 -> {
            val ctx = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(ctx) else dynamicLightColorScheme(ctx)
        }
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }
    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,
        shapes = Shapes,
        content = content
    )
}

private val DarkColorScheme = darkColorScheme(
    primary = ..., secondary = ..., background = ..., onPrimary = ...
)
```

### Цветовые роли

- `primary` / `onPrimary` — основной бренд + текст на нём.
- `secondary` / `onSecondary` — акцент.
- `tertiary` — третичный.
- `error` / `onError`.
- `background` / `onBackground` — общий фон.
- `surface` / `onSurface` — карточки, поверхности.
- `surfaceVariant`, `outline`, `inverseSurface`, etc.

«On» — цвет контента поверх данного цвета (гарантирует контраст).

### Typography

```kotlin
val Typography = Typography(
    displayLarge = TextStyle(fontSize = 57.sp, ...),
    headlineMedium = ...,
    titleLarge = ...,
    bodyMedium = ...,
    labelSmall = ...
)
```

15 ролей в Material 3 (display/headline/title/body/label × large/medium/small).

### Использование

```kotlin
Text(
    "Hello",
    color = MaterialTheme.colorScheme.primary,
    style = MaterialTheme.typography.titleMedium
)
```

### Dark mode (XML)

`res/values/themes.xml`:
```xml
<style name="Theme.App" parent="Theme.Material3.DayNight.NoActionBar">
    <item name="colorPrimary">@color/brand</item>
</style>
```

`res/values-night/themes.xml` — переопределения для тёмной темы.

`res/values/colors.xml` + `res/values-night/colors.xml` для палитры.

В коде:
```kotlin
AppCompatDelegate.setDefaultNightMode(AppCompatDelegate.MODE_NIGHT_FOLLOW_SYSTEM)
```

### Dynamic color (Android 12+)

System извлекает 5 цветов из обоев → tonal palette. Compose: `dynamicLightColorScheme(context)`.

В XML: `?attr/colorPrimary` автоматически берёт из системы, если тема наследует `Theme.Material3.DynamicColors.*`.

```xml
<application android:theme="@style/Theme.Material3.DynamicColors.DayNight">
```

Или programmatic:
```kotlin
DynamicColors.applyToActivitiesIfAvailable(application)
```

### Theme overlay

```xml
<style name="Theme.App.Red" parent="Theme.App">
    <item name="colorPrimary">@color/red</item>
</style>

<!-- применить к одной View -->
<View android:theme="@style/Theme.App.Red" />
```

В Compose:
```kotlin
MaterialTheme(colorScheme = MaterialTheme.colorScheme.copy(primary = Color.Red)) {
    Section()
}
```

### Locale per-app (Android 13+)

```kotlin
AppCompatDelegate.setApplicationLocales(LocaleListCompat.forLanguageTags("ru"))
```

Пользователь может менять язык приложения отдельно от системного.

### Edge-to-edge

```kotlin
enableEdgeToEdge()   // androidx.activity 1.8+
```

Контент за status bar / nav bar. Использовать `WindowInsets` для padding.

```kotlin
Box(Modifier.windowInsetsPadding(WindowInsets.safeDrawing)) { ... }
```

## Подводные камни

- Hardcoded `Color.Red` в Compose → тёмная тема ломается. Использовать `MaterialTheme.colorScheme.error`.
- `values-night` для иконок (vector drawable) — добавляй `?attr/colorOnSurface` вместо `@color/black`.
- Dynamic color на Android < 12 → fallback на ваш `LightColorScheme`/`DarkColorScheme`. Не забыть протестировать.
- `Theme.Material3.*` в XML с Compose — overrides только для AndroidView/legacy.
- Edge-to-edge без `WindowInsets` → контент под gesture nav / status bar.

## Связанные темы

- [[q-19-motion-layout]]
- [[../02-Android-Core/q-14-resources-configuration]]

## Follow-up

- Что такое dynamic color и с какой версии работает?
- Чем `onPrimary` отличается от `primary`?
- Как сделать тёмную тему в Compose?
