---
type: question
category: android-core
difficulty: middle
tags: [manifest, sdk-versions, build]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# AndroidManifest, `minSdk` / `targetSdk` / `compileSdk`. Что они значат?

## Краткий ответ (TL;DR)

- **`compileSdk`** — версия API, против которой компилируется код (определяет, какие классы/методы доступны).
- **`minSdk`** — самая старая версия, на которой приложение установится и заработает.
- **`targetSdk`** — версия, под которую вы протестировали; включает соответствующие behavior changes и компат-режимы.

`compileSdk ≥ targetSdk ≥ minSdk`. Google Play требует **targetSdk не старше чем текущий − 1** (на 2026 — минимум API 34).

## Развёрнутый ответ

### compileSdk

```kotlin
android { compileSdk = 35 }
```

- При `compileSdk = 35` доступны все классы/методы API 35.
- Использование `Build.VERSION.SDK_INT >= 33` нужно для рантайм-проверок при вызове новых API.
- Compile-time только; на старых девайсах эти классы не будут существовать → `NoClassDefFoundError`.

### minSdk

```kotlin
defaultConfig { minSdk = 24 }
```

- Жёсткое отсечение: устройства < minSdk не видят приложение в Play.
- Lint предупредит, если используешь API > minSdk без `@RequiresApi`/version check.
- Поднимая minSdk, вы убиваете часть аудитории, но получаете чистый код без legacy веток.

### targetSdk

```kotlin
defaultConfig { targetSdk = 35 }
```

- Сигнал системе: «я знаю про поведенческие изменения вплоть до этого API».
- Если устройство выше targetSdk — система может применять **compatibility shims** (например, обход некоторых ограничений).
- Каждое поднятие targetSdk → читай **behavior changes**: новые ограничения background, scoped storage, exported для компонентов, foreground service types и т.д.
- **Play Store policy**: новые apps/updates требуют targetSdk не старше N−1 (сейчас API 34, скоро 35).

### Lint и @RequiresApi

```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    notificationChannel()   // O+ API
}
```

`SdkSuppress` / `@RequiresApi` / `@ChecksSdkIntAtLeast` помогают lint понять контекст.

### AndroidManifest минимум

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.x.app">

    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:name=".App"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/AppTheme">

        <activity android:name=".MainActivity" android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

С targetSdk 31+ для каждого компонента с intent-filter **обязателен** `android:exported`.

### Manifest merger

`build.gradle.kts` → итоговый Manifest собирается из:
1. Main manifest.
2. Manifest каждой библиотеки.
3. Build type / flavor overrides.

Конфликты решаются `tools:` атрибутами:
- `tools:replace="android:name"` — заменить.
- `tools:remove="android:foo"` — удалить.
- `tools:node="merge"` / `tools:node="replace"` / `tools:node="remove"`.

Лог мержа: `app/build/outputs/logs/manifest-merger-debug-report.txt`.

### Permissions в манифесте

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.READ_CONTACTS"
                 android:maxSdkVersion="32" />   <!-- только до API 32 -->
```

`<uses-feature>` декларирует hardware-фичу, влияет на видимость в Play:

```xml
<uses-feature android:name="android.hardware.camera" android:required="false" />
```

`required="false"` — устройство без камеры тоже увидит приложение.

### compileSdkPreview / minSdkPreview

Для preview API (например, Android 16 beta) — `compileSdkPreview = "Baklava"`, AGP подхватит из preview SDK.

## Подводные камни

- `targetSdk` стар → Play режет distribution; новые поведенческие защиты не применяются.
- Использование `compileSdk` API на старых устройствах без проверки → crash. Lint ловит, но `@SuppressLint` отключает.
- Manifest merger часто конфликтует с FCM/Crashlytics/etc — лог мержа в помощь.
- Library `minSdk` > app `minSdk` → ошибка сборки. Решение: убрать lib или поднять minSdk.
- `<uses-feature>` забыт → Play покажет приложение на ТВ/часах, где оно не работает.

## Связанные темы

- [[q-04-intents]]
- [[q-11-permissions]]
- [[../06-Build/q-gradle-basics]]

## Follow-up

- Что произойдёт, если запустить API 33 метод на устройстве API 28?
- Почему `targetSdk` влияет на поведение, а не только маркер?
- Когда нужен `tools:replace` в manifest merger?
