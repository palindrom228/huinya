---
type: question
category: android-core
difficulty: middle
tags: [context, leaks]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Что такое Context? `Application`, `Activity`, `Service`, `ContextWrapper`. Когда что использовать?

## Краткий ответ (TL;DR)

`Context` — handle к ресурсам, системным сервисам, файлам, БД, теме приложения. Главные варианты: **`applicationContext`** — живёт пока процесс жив, **`activityContext`** — живёт с Activity, имеет тему и UI-ресурсы. Хранить Activity Context в долгоживущих объектах → утечка Activity.

## Развёрнутый ответ

### Иерархия

```
Context (abstract)
  └─ ContextImpl
       ↑ wrap
  ContextWrapper
       ├─ Application
       ├─ Service
       └─ ContextThemeWrapper
            └─ Activity
```

### Чем отличаются

| Что нужно | Application | Activity |
|-----------|-------------|----------|
| Получить ресурс (`getString`) | ✅ | ✅ |
| Запустить Service | ✅ | ✅ |
| Получить SystemService (`ConnectivityManager`) | ✅ | ✅ |
| Inflate View / получить тему UI | ❌ (themeless) | ✅ |
| Показать `Dialog`/`AlertDialog` | ❌ | ✅ |
| `startActivity` | ✅ с `FLAG_ACTIVITY_NEW_TASK` | ✅ |
| Binding к Service | ⚠️ (для UI лучше Activity) | ✅ |

### Application Context

Singleton процесса. Получить:
- `context.applicationContext`
- `getApplication()` в Activity
- В Hilt — `@ApplicationContext` qualifier.

Безопасен для хранения в долгоживущих объектах (репозитории, синглтоны).

### Activity Context

Содержит тему (`ContextThemeWrapper`) → нужен для `LayoutInflater`, `Dialog`, `View`. Жизненный цикл = Activity. **Утечка Activity Context** = утечка всей View hierarchy.

### Типичные утечки

```kotlin
// 1. Singleton хранит Activity
object Cache { var ctx: Context? = null }
Cache.ctx = this   // ❌ в Activity → утечка

// 2. Inner class неявно держит outer
class MyActivity : Activity() {
    inner class Task : AsyncTask<...>()    // держит ссылку на Activity
}

// 3. static View / Drawable с Activity Context
companion object { var drawable: Drawable? = null }
drawable = ContextCompat.getDrawable(this, R.drawable.x)   // ❌

// 4. Handler в Activity с долгими сообщениями
val handler = Handler(Looper.getMainLooper())
handler.postDelayed({ /* lambda держит Activity */ }, 60_000)
```

### Правило

> «Если объект **переживёт** Activity — давай ему `applicationContext`».

### Когда нужен именно Activity

- Inflate с правильной темой.
- `Dialog` (нельзя на application context — на 26+ бросает `BadTokenException`).
- `WindowManager` для оверлеев конкретного окна.
- `startActivityForResult` (теперь `ActivityResultLauncher`).

### ContextThemeWrapper

Если нужна другая тема для конкретного View:

```kotlin
val themed = ContextThemeWrapper(activity, R.style.Dark)
val view = LayoutInflater.from(themed).inflate(R.layout.x, parent, false)
```

## Подводные камни

- `applicationContext` для inflate → дефолтная тема, не та, что в Activity → внешний вид «не тот».
- `applicationContext.startActivity` без `FLAG_ACTIVITY_NEW_TASK` → exception.
- Хранение Application Context — обычно ок, но через GlobalScope.launch с lambda, ссылающейся на Activity, утечёт всё равно.
- `Context.getColor`/`getDrawable` могут вернуть разные результаты в зависимости от темы → Activity vs Application.

## Связанные темы

- [[q-01-activity-lifecycle]]
- [[../07-Performance-Memory/q-leakcanary]]

## Follow-up

- Почему `Dialog(applicationContext)` упадёт?
- Какой контекст использовать в Hilt-репозитории?
- Что произойдёт, если `static` поле держит `View` из Activity?
