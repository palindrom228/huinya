---
type: category-index
category: data-networking
tags: [index, data-networking]
---

# 6. Data & Networking

Retrofit / OkHttp, Room, DataStore, SharedPreferences, WorkManager, кэширование.

## Подтемы

### Сеть
- OkHttp: interceptors (app vs network), CallAdapter, кэш, connection pool
- Retrofit: converters, suspend, Flow, error handling
- HTTPS, certificate pinning, network security config
- WebSocket, SSE, long-polling
- gRPC (опционально)
- Кэширование (HTTP cache headers, etag), offline-first

### Persistence
- Room: entities, DAO, relations, migrations, multi-process
- DataStore (Preferences vs Proto), миграция с SharedPreferences
- SharedPreferences проблемы (sync, ANR)
- SQLite напрямую vs Room
- Encrypted storage (EncryptedSharedPreferences, SQLCipher)
- File I/O, scoped storage

### Background work
- WorkManager: constraints, retry, chaining, periodic, expedited
- AlarmManager, JobScheduler
- Foreground services vs WorkManager
- Sync adapters (legacy)

## Вопросы

```dataview
TABLE WITHOUT ID
  file.link AS "Вопрос",
  difficulty AS "Сложность",
  status AS "Статус",
  times_failed AS "Провалов"
FROM "01-Questions/06-Data-Networking"
WHERE type = "question"
SORT difficulty DESC, status ASC
```
