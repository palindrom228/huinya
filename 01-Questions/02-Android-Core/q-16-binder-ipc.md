---
type: question
category: android-core
difficulty: senior
tags: [binder, ipc, aidl]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Binder и IPC: как работает, ограничения, AIDL/Messenger

## Краткий ответ (TL;DR)

**Binder** — Linux kernel driver + Android-фреймворк для **IPC между процессами**. Используется везде: вызов в системные сервисы (ActivityManager, PowerManager), bound services, ContentProvider, AppWidget. Лимит транзакции — **1MB на процесс** (общий буфер). Превышение — `TransactionTooLargeException`. Опции: AIDL (типизированный), Messenger (queue), File descriptors.

## Развёрнутый ответ

### Архитектура

```
App process              kernel               Service process
  ┌──────┐                                     ┌──────┐
  │ Proxy│──→ transact ──→ /dev/binder ──→ ──→│ Stub │
  │      │←── reply  ←──                  ←──│      │
  └──────┘                                     └──────┘
```

- В каждом процессе есть **Binder thread pool** (по умолчанию ~16 потоков).
- Вызов IPC → блокирует вызывающий поток до ответа (если не oneway).
- На сервере вызов диспатчится на binder-поток.

### Лимиты

- **Transaction buffer**: ~1 MB **на весь процесс** (общий, не на вызов).
- Большие данные → `Bundle`/`Parcel` упасть с `TransactionTooLargeException`.
- Решения: разбить на куски, использовать `ParcelFileDescriptor`/shared memory, MemoryFile, AshmemUnix domain sockets.
- Async transactions (`FLAG_ONEWAY`) тоже считаются в буфере.

### AIDL

```aidl
// IRemote.aidl
package com.x;
interface IRemote {
    int add(int a, int b);
    oneway void log(String msg);   // fire-and-forget
}
```

```kotlin
class RemoteService : Service() {
    private val binder = object : IRemote.Stub() {
        override fun add(a: Int, b: Int): Int = a + b
        override fun log(msg: String) { /* ... */ }
    }
    override fun onBind(intent: Intent): IBinder = binder
}
```

Клиент:

```kotlin
val conn = object : ServiceConnection {
    override fun onServiceConnected(name: ComponentName, service: IBinder) {
        val remote = IRemote.Stub.asInterface(service)
        val sum = remote.add(1, 2)   // блокирующий IPC
    }
    override fun onServiceDisconnected(name: ComponentName) {}
}
bindService(intent, conn, BIND_AUTO_CREATE)
```

### Messenger

Высокоуровневая обёртка над AIDL для **очереди сообщений** (без потока на каждый вызов):

```kotlin
val messenger = Messenger(Handler(Looper.getMainLooper()) { msg ->
    handle(msg)
    true
})

val remoteMessenger = Messenger(binder)
remoteMessenger.send(Message.obtain(null, MSG_CMD, data))
```

Удобно для one-way callbacks. Один поток — нет race conditions.

### Bound to system services

```kotlin
val connectivity = context.getSystemService<ConnectivityManager>()!!
connectivity.activeNetwork    // IPC к system_server
```

Многие безобидные вызовы — на самом деле IPC. Дёрни 1000 раз в цикле → видимый jank.

### linkToDeath

```kotlin
binder.linkToDeath(object : IBinder.DeathRecipient {
    override fun binderDied() { /* пересоздать соединение */ }
}, 0)
```

Реагирует на смерть удалённого процесса.

### Безопасность

- **`getCallingUid()` / `getCallingPid()`** — UID/PID вызывающего процесса (для permission check).
- **`enforceCallingPermission(perm)`** — выбросит `SecurityException`, если у клиента нет permission.
- **`clearCallingIdentity()`** — временно сменить identity на свой (для подвызовов).

### Производительность

- IPC дороже локального вызова в ~10–100 раз (микросекунды vs наносекунды).
- Большие Bundle → marshalling overhead + binder buffer pressure.
- Хорошие практики: бактчить вызовы, кэшировать результаты, async (`oneway`) где возможно.

## Подводные камни

- `TransactionTooLargeException` ловит много сразу — обычно после череды Bundle, переполняется буфер.
- AIDL генерит код в kotlin/java — но в gradle нужно `aidl { ... }` в module config.
- `oneway` методы возвращают void немедленно, но имеют **очередь на ~16 transactions** на стороне target — переполнение → drop.
- Bind с разными процессами требует `android:process=":remote"` или другой `applicationId`.
- Cursor из ContentProvider держит binder reference → не закрытый → утечка.

## Связанные темы

- [[q-04-intents]]
- [[q-06-services]]
- [[q-12-content-provider]]

## Follow-up

- Что значит `TransactionTooLargeException` и как её избежать?
- Чем `oneway` метод в AIDL отличается от обычного?
- Почему вызов `connectivity.activeNetwork` в hot path может тормозить?
