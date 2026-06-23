---
type: question
category: android-core
difficulty: middle
tags: [intent, ipc]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Intent: explicit vs implicit, flags, PendingIntent. Безопасность.

## Краткий ответ (TL;DR)

**Explicit** intent — указан конкретный component (своя Activity/Service). **Implicit** — action + data, система ищет подходящий handler. Flags управляют task/back stack. **`PendingIntent`** — токен, передаваемый другому процессу для запуска чего-то от твоего имени (notification, alarm, widget). С Android 12+ требуется явный `FLAG_IMMUTABLE`/`FLAG_MUTABLE`.

## Развёрнутый ответ

### Explicit

```kotlin
startActivity(Intent(this, DetailActivity::class.java).apply {
    putExtra("id", 42L)
})
```

Используется для внутренней навигации. Безопасно — никто другой не перехватит.

### Implicit

```kotlin
val intent = Intent(Intent.ACTION_VIEW, Uri.parse("https://example.com"))
startActivity(intent)
```

Система показывает chooser или открывает default handler. На Android 11+ есть **package visibility**: чтобы видеть, какие приложения могут обработать intent, нужен `<queries>` блок в манифесте.

```xml
<queries>
    <intent>
        <action android:name="android.intent.action.VIEW" />
        <data android:scheme="https" />
    </intent>
</queries>
```

### Intent Flags (для Activity)

| Flag | Эффект |
|------|--------|
| `FLAG_ACTIVITY_NEW_TASK` | Открыть в новой task (нужно для startActivity из non-Activity Context) |
| `FLAG_ACTIVITY_CLEAR_TOP` | Если Activity уже в стеке — убрать всё, что сверху |
| `FLAG_ACTIVITY_SINGLE_TOP` | Если Activity на вершине — не создавать новую, доставить через `onNewIntent` |
| `FLAG_ACTIVITY_CLEAR_TASK` + `NEW_TASK` | Очистить task и стартовать сначала (logout) |
| `FLAG_ACTIVITY_NO_HISTORY` | Не сохранять в back stack |

### launchMode (в манифесте)

- `standard` — каждая новая instance.
- `singleTop` — если на вершине → `onNewIntent`.
- `singleTask` — только одна instance в системе, при повторе → `onNewIntent` + очистка сверху.
- `singleInstance` — одна instance в собственной task.

### PendingIntent

«Право выполнить что-то от моего имени». Используется в:
- Notifications (`setContentIntent`, action button)
- AlarmManager
- AppWidget
- Slice/Tiles

```kotlin
val pi = PendingIntent.getActivity(
    context, 0, intent,
    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
)
```

Android 12+ требует **обязательно** указать `FLAG_IMMUTABLE` или `FLAG_MUTABLE`. Mutable нужен для inline reply в notifications, slices — иначе всегда immutable.

Флаги обновления:
- `FLAG_UPDATE_CURRENT` — обновить extras существующего PI.
- `FLAG_CANCEL_CURRENT` — отменить старый и создать новый.
- `FLAG_NO_CREATE` — вернуть `null` если не существует.

### Безопасность intents

1. **Implicit intent для confidential data** → его может перехватить любое приложение → используй explicit.
2. **`exported` для компонентов**: с Android 12+ обязателен явный `android:exported`. Если компонент имеет intent-filter → exported по умолчанию true до Android 12.
3. **Intent redirection**: если твой component получает Intent с extras, содержащим другой Intent → не передавай его системе без валидации (CVE-классика).
4. **PendingIntent + mutable + implicit base** → атакующий может подменить компонент. Лучше immutable + explicit.

### onActivityResult → ActivityResultLauncher

```kotlin
val launcher = registerForActivityResult(ActivityResultContracts.GetContent()) { uri ->
    uri?.let { handle(it) }
}
launcher.launch("image/*")
```

Преимущества:
- Type-safe contracts.
- Регистрируется в `onCreate` (до `STARTED`), не зависит от `requestCode`.
- Корректно работает при смерти процесса (state сохраняется).

## Подводные камни

- `startActivity` из не-Activity (Service, Receiver) без `FLAG_ACTIVITY_NEW_TASK` → exception.
- Implicit intent на Android 11+ без `<queries>` → `resolveActivity()` вернёт null.
- `PendingIntent` с одним и тем же `requestCode` и `equals`-эквивалентным intent → переиспользует существующий (extras не обновятся без `FLAG_UPDATE_CURRENT`).
- Передача >1MB через extras → `TransactionTooLargeException`.

## Связанные темы

- [[q-05-process-and-task]]
- [[../11-Security/q-component-export]]

## Follow-up

- Почему с Android 12 PendingIntent нужно явно помечать immutable/mutable?
- Чем `singleTop` отличается от `singleTask`?
- Что произойдёт, если запустить Activity из Service без `FLAG_ACTIVITY_NEW_TASK`?
