---
type: question
category: android-core
difficulty: senior
tags: [touch, looper, handler, choreographer]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Looper, Handler, MessageQueue, Choreographer. Как работает main thread?

## Краткий ответ (TL;DR)

Main thread имеет `Looper`, который крутит цикл, забирая `Message` из `MessageQueue` и доставляя в `Handler.handleMessage`. Touch events, system callbacks, ваши `post`-задачи — всё через эту очередь. `Choreographer` синхронизирует frame callbacks с VSYNC (~16.6ms на 60Hz, 8.3ms на 120Hz). Превысил frame budget — дроп кадра.

## Развёрнутый ответ

### Looper

```kotlin
class MyThread : Thread() {
    override fun run() {
        Looper.prepare()
        val handler = Handler(Looper.myLooper()!!) { msg -> handle(msg); true }
        Looper.loop()    // блокирует поток
    }
}
```

`Looper.prepare()` создаёт `MessageQueue` для текущего потока. `Looper.loop()` — бесконечный цикл `next() → dispatchMessage()`.

Main thread у Android уже имеет Looper (создан в `ActivityThread.main`).

### MessageQueue

Сортирована по `when` (execute time). `Message` имеет `target` (handler) и `callback` (Runnable). Используется priority queue с поддержкой:
- **Barriers** — блокируют синхронные сообщения, пропускают только asynchronous.
- **Asynchronous messages** — обходят barrier (используется Choreographer для frame callbacks → input не блокируется тяжёлыми post'ами).

### Handler

```kotlin
val handler = Handler(Looper.getMainLooper())
handler.post { updateUi() }
handler.postDelayed({ ... }, 1000)
handler.postAtFrontOfQueue { ... }   // обходит очередь
```

#### Утечки Handler

```kotlin
class MyActivity : Activity() {
    val handler = Handler(Looper.getMainLooper())
    override fun onResume() {
        handler.postDelayed({ updateUi() }, 60_000)   // лямбда держит Activity
    }
}
```

Если Activity закрыта, а сообщение ещё в очереди → Activity не GC. Решения: `handler.removeCallbacksAndMessages(null)` в `onDestroy`, или использовать Coroutines с lifecycle scope.

### Choreographer

Drawing pipeline:

```
VSYNC → Choreographer.doFrame()
  ├─ INPUT callbacks (handleInputEvents)
  ├─ ANIMATION callbacks (Animator, Compose recomposer)
  ├─ INSETS callbacks
  ├─ TRAVERSAL callbacks (measure → layout → draw)
  └─ COMMIT (sync to RenderThread, displayList)
```

```kotlin
Choreographer.getInstance().postFrameCallback { frameTimeNanos ->
    // вызывается на следующем VSYNC
}
```

### Frame budget

| Refresh rate | Budget |
|-------------|--------|
| 60 Hz | 16.6 ms |
| 90 Hz | 11.1 ms |
| 120 Hz | 8.3 ms |

Превышение → **jank** (пропуск кадра). Google Play отслеживает frozen frames (>700ms).

### Input events

1. Драйвер → `/dev/input/eventX`.
2. `InputManagerService` → `InputDispatcher`.
3. `ViewRootImpl.WindowInputEventReceiver` (async message).
4. View hierarchy: `dispatchTouchEvent → onInterceptTouchEvent → onTouchEvent`.

`onTouchEvent` → `GestureDetector` → ваши listeners.

### Coroutines on main

`Dispatchers.Main` под капотом использует `Handler(Looper.getMainLooper())` для `post`. `Dispatchers.Main.immediate` — если уже на main, не post'ит, выполняет синхронно (нужно для `setValue`-семантики и оптимизаций StateFlow в UI).

### Тестирование

- `Robolectric` — свой Looper, можно `shadowOf(Looper.getMainLooper()).idle()`.
- `kotlinx-coroutines-test` — `TestDispatcher`, `runTest`.
- Espresso ждёт idle через `IdlingResource` — синхронизируется с Looper.

## Подводные камни

- `Handler()` без аргументов — deprecated, использует looper текущего потока (может быть не тот).
- `Handler(myCallback)` забывает указать looper → null.
- Post тяжёлой работы → блокирует frame → jank.
- `runOnUiThread { ... }` если уже на UI → выполняется синхронно, иначе post.
- `removeCallbacks(runnable)` — сравнивает по `==`, лямбда без сохранённой ссылки не удалится.

## Связанные темы

- [[q-10-anr]]
- [[../07-Performance-Memory/q-jank]]
- [[../09-Compose/q-recomposition]]

## Follow-up

- Чем `Dispatchers.Main.immediate` отличается от `Dispatchers.Main`?
- Почему long-running `post` на main блокирует touch?
- Как Choreographer гарантирует, что input обрабатывается до анимации?
