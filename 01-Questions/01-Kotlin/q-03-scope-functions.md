---
type: question
category: kotlin
difficulty: middle
tags: [kotlin, idioms]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Чем отличаются `let`, `run`, `with`, `apply`, `also`? Когда какую выбрать?

## Краткий ответ (TL;DR)

Различия в двух осях: **получатель** (`it` vs `this`) и **возвращаемое значение** (объект vs результат лямбды).

| Функция | Получатель | Возвращает | Типичный кейс |
|---------|------------|------------|---------------|
| `let`   | `it`  | результат лямбды | null-check, scope для nullable, преобразование |
| `run`   | `this`| результат лямбды | конфигурация + вычисление значения |
| `with`  | `this`| результат лямбды | группировка вызовов на готовом объекте |
| `apply` | `this`| сам объект | builder, конфигурация объекта |
| `also`  | `it`  | сам объект | side-effect (логирование), без shadowing `this` |

## Развёрнутый ответ

### Правила выбора

1. **Возвращаешь объект?** → `apply` / `also`.
2. **Возвращаешь результат?** → `let` / `run` / `with`.
3. **Нужно избежать shadowing `this`** (например, внутри Activity/Fragment) → `let` / `also` (через `it`).
4. **Конфигурируешь и возвращаешь объект** → `apply` (builder-style).
5. **Side effect без логики** → `also`.

### Примеры

```kotlin
// apply — builder
val intent = Intent(this, MainActivity::class.java).apply {
    putExtra("id", 42)
    addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
}

// also — side effect
val users = repo.loadUsers()
    .also { Log.d("Users", "got ${it.size}") }

// let — null safety
user?.let { sendEmail(it.email) }

// run — конфиг + значение
val area = Rectangle(10, 20).run {
    width * height
}

// with — группировка вызовов
with(canvas) {
    drawLine(...)
    drawRect(...)
    drawCircle(...)
}
```

## Подводные камни

- `let` на nullable возвращает `null`, если получатель `null` — учитывай в цепочках.
- `apply` внутри Activity легко затеняет `this` Activity → ошибки с listener'ами.
- Чрезмерное вложение scope-функций → нечитаемый код. Лучше `val` с осмысленным именем.

## Связанные темы

- [[q-02-null-safety]]
- [[q-06-delegates]]

## Follow-up

- Чем `with(x) { ... }` отличается от `x.run { ... }`?
- Почему `apply` хорош для DI/конфигурации, а `also` — для логов?
