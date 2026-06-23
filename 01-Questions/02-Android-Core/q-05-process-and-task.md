---
type: question
category: android-core
difficulty: senior
tags: [process, task, backstack]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Process, Application, Task, Back Stack — как они соотносятся

## Краткий ответ (TL;DR)

**Process** — Linux-процесс, в котором живёт твой код (один процесс на приложение по умолчанию, можно несколько через `android:process`). **Application** — singleton-объект, по одному на процесс. **Task** — стек Activity, единица в Recents. **Back Stack** — стек Activity внутри task. Одно приложение может иметь несколько task, одна task может содержать Activity из разных приложений.

## Развёрнутый ответ

### Process

- По умолчанию все компоненты одного `applicationId` запускаются в одном процессе.
- `android:process=":remote"` — приватный процесс с тем же UID; `android:process="com.x.shared"` — расшариваемый между приложениями (требует одинаковую подпись).
- Каждый процесс — отдельный JVM, отдельный heap, **отдельный `Application` объект** (его `onCreate` вызовется в каждом процессе!).
- Лимиты heap зависят от device (32–512MB+).

### Application

```kotlin
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        // вызывается в КАЖДОМ процессе приложения
        if (currentProcessName() == packageName) initMain() else initWorker()
    }
}
```

Один экземпляр на процесс. Не путать с понятием «приложение в Play Store».

### Task

Task — последовательность Activity, с которой пользователь взаимодействует как с одной задачей. Видна в Recents как одна карточка.

```
Task (id=42)
  ├─ MainActivity      ← root
  ├─ ListActivity
  └─ DetailActivity    ← top
```

При нажатии Back DetailActivity убирается из стека, при удалении из Recents — task убивается.

### Когда новая Task

- `FLAG_ACTIVITY_NEW_TASK` (+ `taskAffinity` определяет, в какую task попадёт).
- `launchMode="singleTask"` или `"singleInstance"`.
- Запуск с другого `taskAffinity`.

### taskAffinity

Атрибут Activity (или Application — для всех Activity по умолчанию):

```xml
<activity android:name=".A" android:taskAffinity="com.x.taskA" />
<activity android:name=".B" android:taskAffinity="com.x.taskB" />
```

При `FLAG_ACTIVITY_NEW_TASK` система ищет task с подходящим affinity. По умолчанию affinity = `applicationId`.

### Back Stack — особенности

- `finish()` убирает текущую Activity.
- `finishAffinity()` убирает всю цепочку с тем же affinity.
- При config change Activity пересоздаётся, но позиция в стеке сохраняется.
- Прозрачные Activity не «прячут» нижнюю — нижняя в `onPause`, не `onStop`.

### Process priority (oom_adj)

| Уровень | Кто | Шанс убить |
|---------|-----|------------|
| Foreground | Активная Activity, foreground Service, видимый IME | минимальный |
| Visible | Activity в `onPause` (виден сверху диалог), bound к visible | низкий |
| Service | Запущенный обычный Service | средний |
| Cached/Background | Активности в фоне, кэшированные процессы | высокий — убиваются первыми |
| Empty | Процесс без активных компонентов (cache) | очень высокий |

### LRU и kill

Когда система нуждается в памяти, она убивает процессы по приоритету + LRU. Backgrounded app может быть убит в любой момент → state нужно сохранять (см. `onSaveInstanceState`, `SavedStateHandle`).

## Подводные камни

- `Application.onCreate()` в multi-process инициализирует **всё** в каждом процессе → запуск может тормозить. Проверяй `currentProcessName`.
- WorkManager/Firebase в `:remote` процессе → удвоение init, дубли работы.
- `taskAffinity` — мощный, но легко создать «потерянные» Activity-карточки в Recents.
- `singleInstance` Activity в своей task → запускаемые из неё через `startActivity` уходят в **другую** task.

## Связанные темы

- [[q-04-intents]]
- [[q-09-process-death]]
- [[q-06-services]]

## Follow-up

- Почему `Application.onCreate()` может вызваться несколько раз?
- Чем `singleTask` отличается от `singleInstance` с точки зрения back stack?
- Что произойдёт с заданием WorkManager, если процесс убит?
