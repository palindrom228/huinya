---
type: category-index
category: android-core
tags: [index, android-core]
---

# 2. Android Core

Жизненные циклы, компоненты, Context, Intent, Manifest, процессы, IPC.

## Подтемы

- Activity / Fragment / Service / BroadcastReceiver / ContentProvider lifecycles
- Configuration changes, `ViewModel` survival, `SavedStateHandle`
- Context: типы, утечки, когда какой использовать
- Intent / PendingIntent, flags, deeplinks, App Links
- Processes & threads, multi-process apps
- IPC: AIDL, Messenger, ContentProvider, Binder
- Foreground/Background services, restrictions (Doze, App Standby)
- Notifications, channels, importance
- App startup, Application class
- Storage Access Framework, scoped storage

## Вопросы

```dataview
TABLE WITHOUT ID
  file.link AS "Вопрос",
  difficulty AS "Сложность",
  status AS "Статус",
  times_failed AS "Провалов"
FROM "01-Questions/02-Android-Core"
WHERE type = "question"
SORT difficulty DESC, status ASC
```
