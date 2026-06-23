---
type: question
category: android-core
difficulty: middle
tags: [permissions, runtime, scoped]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Permissions: install-time vs runtime. Запрос, rationale, "Don't ask again". Scoped storage, MediaStore.

## Краткий ответ (TL;DR)

**Install-time** (`normal`, `signature`) выдаются автоматически. **Runtime** (`dangerous`) нужно запрашивать у пользователя в рантайме (с API 23). Use cases: рекомендуется показывать **rationale** перед запросом. С Android 11+ если пользователь дважды отказал — система больше не показывает диалог. С Android 10+ — **scoped storage**, доступ к файлам только через MediaStore/SAF.

## Развёрнутый ответ

### Уровни protection

- **normal** — INTERNET, ACCESS_NETWORK_STATE — выдаются автоматически.
- **signature** — приложения с той же подписью, что и declarer.
- **dangerous** — runtime: CAMERA, LOCATION, READ_CONTACTS, RECORD_AUDIO.
- **special** — отдельный поток (`Settings.canDrawOverlays`, `MANAGE_EXTERNAL_STORAGE`).

### Запрос runtime permission

Современный способ — ActivityResultLauncher:

```kotlin
val permLauncher = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { granted ->
    if (granted) startCamera() else showDenied()
}

// несколько сразу
val multi = registerForActivityResult(
    ActivityResultContracts.RequestMultiplePermissions()
) { results -> ... }

fun onClickCamera() {
    when {
        ContextCompat.checkSelfPermission(this, CAMERA) == PERMISSION_GRANTED -> startCamera()
        shouldShowRequestPermissionRationale(CAMERA) -> showRationaleDialog()
        else -> permLauncher.launch(CAMERA)
    }
}
```

### shouldShowRequestPermissionRationale

Возвращает `true`, если:
- Пользователь раньше отказал, но НЕ выбрал «Don't ask again».
- Это **сигнал**, что нужно объяснить, зачем permission.

Возвращает `false`:
- Никогда не запрашивали.
- Отказали с «Don't ask again» / Android 11+ дважды.
- Запрет policy.

Различать «никогда не спрашивали» и «отказали навсегда» одним методом нельзя — нужно хранить флаг «спрашивали ли раньше» в prefs.

### Android 11+: автоматический «don't ask»

Если пользователь дважды отказал — система больше не покажет диалог даже при вызове `launch`. Нужно вести в **Settings → App permissions** через intent:

```kotlin
val intent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS,
    Uri.fromParts("package", packageName, null))
startActivity(intent)
```

### One-time permission (Android 11+)

Пользователь может выдать **«Only this time»** для LOCATION, MIC, CAMERA. После kill процесса — снова попросят. Это прозрачно для кода — просто помни, что permission может «исчезнуть».

### Scoped storage (Android 10+)

- App-specific files (`context.filesDir`, `getExternalFilesDir`) — без permissions.
- MediaStore (фото/видео/аудио общие): чтение своих файлов — без permission, чужих — с `READ_MEDIA_*` (Android 13+) или `READ_EXTERNAL_STORAGE`.
- Произвольные файлы — через SAF (`ACTION_OPEN_DOCUMENT`), без permissions.
- `MANAGE_EXTERNAL_STORAGE` — особый, доступ ко всему storage, требуется обоснование на Play Console.

### Android 13+: гранулярные media permissions

Вместо `READ_EXTERNAL_STORAGE`:
- `READ_MEDIA_IMAGES`
- `READ_MEDIA_VIDEO`
- `READ_MEDIA_AUDIO`

Android 14+:
- `READ_MEDIA_VISUAL_USER_SELECTED` — partial access (пользователь выбирает только некоторые фото).

### Background location

`ACCESS_BACKGROUND_LOCATION` запрашивается **отдельно**, после того как пользователь дал foreground. На Android 11+ — только через системные настройки, нельзя в одном диалоге.

### Notification permission (Android 13+)

`POST_NOTIFICATIONS` — runtime. Без неё уведомления не показываются. Нужно запросить при первом нужном моменте.

## Подводные камни

- `shouldShowRequestPermissionRationale` не различает «впервые» и «навсегда» — нужен свой флаг.
- В тестах permission может выдаваться через `GrantPermissionRule`.
- Permission снимается при изменении настроек → проверяй `checkSelfPermission` перед каждым использованием.
- `MANAGE_EXTERNAL_STORAGE` — Play Store policy запрещает использовать его без чёткого обоснования, иначе reject.
- На foldables/multi-window permission state не меняется при config change.

## Связанные темы

- [[../05-Storage/q-scoped-storage]]
- [[../11-Security/q-permissions-best-practices]]

## Follow-up

- Как отличить «впервые спрашиваем» от «отказали навсегда»?
- Почему `READ_EXTERNAL_STORAGE` не нужен для чтения своих фото?
- Что произойдёт, если в Android 13+ не запросить POST_NOTIFICATIONS?
