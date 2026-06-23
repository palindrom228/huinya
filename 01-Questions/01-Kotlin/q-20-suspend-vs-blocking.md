---
type: question
category: kotlin
difficulty: senior
tags: [kotlin, coroutines, performance]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# `suspend` vs blocking vs async: что значит «main-safe» и почему `withContext(IO)` не везде нужен

## Краткий ответ (TL;DR)

`suspend` — маркер «может приостановиться, не блокирует поток». Но **сам по себе** не делает функцию неблокирующей: если внутри `Thread.sleep` — поток встанет. **Main-safe** функция — гарантирует, что не заблокирует вызывающий поток. Ответственность — на нижнем слое (репозиторий/data source), а не на ViewModel.

## Развёрнутый ответ

### Три категории функций

| Тип | Пример | Поведение |
|-----|--------|-----------|
| Blocking | `Thread.sleep`, JDBC, `inputStream.read` | Останавливает поток на время операции |
| Suspending | `delay`, Retrofit suspend, Room suspend DAO | Освобождает поток, resume через callback |
| Async-callback | `OkHttp.enqueue(callback)` | Не блокирует, но callback-стиль |

### Что такое main-safe

Функция main-safe, если её **безопасно вызывать на Main thread** — она не заблокирует UI. Например:

```kotlin
// ❌ НЕ main-safe — несмотря на suspend
suspend fun loadFromDisk(): String {
    return File("...").readText()    // блокирует поток!
}

// ✅ main-safe
suspend fun loadFromDisk(): String = withContext(Dispatchers.IO) {
    File("...").readText()
}
```

### Главное правило (Google guide)

**Suspend-функции должны быть main-safe.** Это значит — ответственность переключения контекста лежит на **функции, которая делает блокирующую работу**, а не на её вызывателе.

```kotlin
// Repository — переключает себя
class UserRepository(private val dao: UserDao) {
    suspend fun load(id: Long): User = withContext(Dispatchers.IO) {
        dao.loadBlocking(id)
    }
}

// ViewModel — не думает о Dispatchers
viewModelScope.launch {
    val u = repo.load(id)   // безопасно из Main
    _state.value = u
}
```

### Когда withContext НЕ нужен

Если API уже suspend и main-safe — лишний `withContext` создаёт ненужный context switch:

```kotlin
// Retrofit suspend сам main-safe (внутри своего dispatcher'а)
suspend fun loadUser() = api.getUser()   // НЕ нужно withContext(IO)

// Room suspend DAO тоже main-safe
suspend fun load() = dao.getAll()
```

### IO vs Default

- **IO** — для **блокирующего IO** (File, JDBC, blocking OkHttp). Пул может расти до 64.
- **Default** — для **CPU-bound** (parsing, sorting, image processing). Пул = max(2, cores).

```kotlin
suspend fun process(json: String): User = withContext(Dispatchers.Default) {
    parseJsonExpensive(json)
}
```

### Распространённые ошибки

```kotlin
// 1. withContext(IO) поверх suspend Retrofit — бесполезный switch
suspend fun get() = withContext(Dispatchers.IO) { api.getSuspend() }   // ❌

// 2. async + immediate await — лишний оверхед
val u = async { api.getUser() }.await()                                // ❌
val u = api.getUser()                                                  // ✅

// 3. async без structured concurrency (GlobalScope)
GlobalScope.async { api.getUser() }                                    // ❌

// 4. runBlocking в suspend контексте → блокирует поток
suspend fun bad() = runBlocking { api.get() }                          // ❌
```

### parallel decomposition

```kotlin
suspend fun loadAll(): Profile = coroutineScope {
    val userDef = async { api.user() }
    val postsDef = async { api.posts() }
    Profile(userDef.await(), postsDef.await())
}
```

`coroutineScope` гарантирует: при ошибке одного — отменяется другой; родитель не завершится, пока оба не закончатся.

## Подводные камни

- `suspend` не магия: блокирующий код внутри = блокировка потока.
- Лишний `withContext` = context switch (≈ микросекунды + аллокации).
- `runBlocking` в Android — почти всегда антипаттерн (только в `main` функции консольного приложения / тестах).
- `Dispatchers.IO` пул общий для процесса — забить его длинными blocking операциями → starvation других.

## Связанные темы

- [[q-10-coroutines-basics]]
- [[q-11-coroutine-context]]

## Follow-up

- Почему `withContext(Dispatchers.IO)` поверх suspend Retrofit избыточен?
- Когда выбрать `Default` vs `IO`?
- Сколько потоков в `Dispatchers.IO` по умолчанию?
