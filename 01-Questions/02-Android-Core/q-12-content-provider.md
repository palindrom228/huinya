---
type: question
category: android-core
difficulty: middle
tags: [contentprovider, ipc, fileprovider]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# ContentProvider: зачем нужен, FileProvider, безопасный шаринг файлов

## Краткий ответ (TL;DR)

`ContentProvider` — компонент для **расшаривания структурированных данных** между приложениями через URI (`content://authority/path`). Сегодня редко используется напрямую — внутри приложения предпочитают репозитории. Главные актуальные кейсы: **FileProvider** для безопасного шаринга файлов (вместо `file://`), интеграции с системой (Contacts, MediaStore), CMS-приложения.

## Развёрнутый ответ

### Зачем существует

С Android 7+ передача `file://` URI между приложениями запрещена (`FileUriExposedException`). `ContentProvider` даёт абстракцию: URI + permission model + автоматическая grant'ация чтения/записи через `FLAG_GRANT_READ_URI_PERMISSION`.

### FileProvider

Сценарий: ваше приложение хочет дать другому (камера, share) доступ к своему файлу.

```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

```xml
<!-- res/xml/file_paths.xml -->
<paths>
    <files-path name="my_images" path="images/" />
    <cache-path name="cache_images" path="images/" />
    <external-files-path name="ext_images" path="images/" />
</paths>
```

```kotlin
val uri = FileProvider.getUriForFile(context, "${packageName}.fileprovider", file)
val share = Intent(Intent.ACTION_SEND).apply {
    type = "image/jpeg"
    putExtra(Intent.EXTRA_STREAM, uri)
    addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
}
startActivity(Intent.createChooser(share, null))
```

### Свой ContentProvider

Нужен редко: только если **внешние приложения** должны читать ваши данные.

```kotlin
class NotesProvider : ContentProvider() {
    override fun onCreate(): Boolean = true  // вызывается ОЧЕНЬ рано, ДО Application.onCreate? — НЕТ, после
    override fun query(uri: Uri, ...): Cursor? = ...
    override fun insert(uri: Uri, values: ContentValues?): Uri? = ...
    override fun update(...) = ...
    override fun delete(...) = ...
    override fun getType(uri: Uri): String? = ...
}
```

⚠️ Поправка: `ContentProvider.onCreate` вызывается **ПОСЛЕ** `Application.attachBaseContext`, но **ДО** `Application.onCreate`. Поэтому он часто используется для **bootstrap инициализации без явного вызова** — например, AndroidX Startup, WorkManager, Firebase используют этот трюк.

### App Startup

Старый паттерн: каждая библиотека объявляла свой `ContentProvider` для авто-init. Это плодило провайдеров и тормозило старт.

Современный подход — **AndroidX Startup**:

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data android:name="com.x.MyInitializer"
               android:value="androidx.startup" />
</provider>
```

Один провайдер запускает все инициализаторы (с зависимостями), много быстрее.

### Безопасность

- `android:exported="true"` без permission → любое приложение может читать/писать.
- `android:permission="..."`, `readPermission`, `writePermission` — фильтр.
- `grantUriPermissions="true"` + `FLAG_GRANT_READ_URI_PERMISSION` в Intent — гранулярная выдача доступа к конкретному URI.
- **Path traversal**: при работе с пользовательскими путями валидируй `File.canonicalPath` — иначе клиент через `../../etc/passwd` может вытащить чужие файлы.

### MediaStore vs FileProvider vs SAF

| | MediaStore | FileProvider | SAF |
|---|------------|--------------|-----|
| Что | Системная БД медиа | Шаринг своих файлов | Пользовательский picker |
| Permissions | `READ_MEDIA_*` | grant flags | нет |
| Видимость | Все media | Только то, что отдали | Что выбрал юзер |
| Когда | Свои фото в галерею | Send/Open в другую app | Open/Save в любое место |

## Подводные камни

- `file://` URI с Android 7+ → `FileUriExposedException` при передаче в Intent.
- `FileProvider` без `<meta-data>` или без правильных путей → `IllegalArgumentException`.
- `ContentProvider.onCreate` тяжёлый → app startup тормозит.
- Multi-process app: provider в каждом процессе, если не указан `android:multiprocess` или process.
- Cursor нужно закрывать (`use { }` или `cursor.close()`).

## Связанные темы

- [[q-04-intents]]
- [[../05-Storage/q-scoped-storage]]
- [[../07-Performance-Memory/q-startup]]

## Follow-up

- Почему с Android 7+ нельзя передавать `file://` URI?
- Как библиотеки используют ContentProvider для auto-init?
- Чем `FileProvider` отличается от MediaStore?
