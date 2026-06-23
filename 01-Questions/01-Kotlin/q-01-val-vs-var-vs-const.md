---
type: question
category: kotlin
difficulty: junior
tags: [kotlin, basics]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# В чём разница между `val`, `var` и `const val`?

## Краткий ответ (TL;DR)

- `var` — mutable reference, можно переприсваивать.
- `val` — read-only reference (не immutable объект!), присваивается один раз.
- `const val` — compile-time константа: только примитивы и `String`, только на top-level или в `object`/`companion`, инлайнится в bytecode.

## Развёрнутый ответ

**`val` ≠ immutable.** Это read-only ссылка. Сам объект может быть mutable:

```kotlin
val list = mutableListOf(1, 2, 3)
list.add(4) // OK — мутируем содержимое
// list = mutableListOf() // ❌ ошибка компиляции
```

**`val` может быть с custom getter** — значит, при каждом обращении может возвращать разное:

```kotlin
val now: Long get() = System.currentTimeMillis()
```

**`const val`** — известно компилятору во время компиляции. Используется в аннотациях, инлайнится по месту использования (бинарная совместимость: изменил `const` → нужно пересобрать всех клиентов).

```kotlin
const val API_VERSION = "v3"   // OK
const val PI = 3.14            // OK
// const val LIST = listOf(1)  // ❌ нельзя
```

## Подводные камни

- `val` в data class — backing field создаётся; в interface — только getter.
- Изменение `const val` ломает бинарную совместимость (значение зашито у клиента).
- В Java `val` виден как `final`, но без `@JvmField` геттер обязателен.

## Связанные темы

- [[q-04-data-vs-value-vs-sealed]]
- [[q-15-jvm-interop]]

## Follow-up

- Можно ли сделать `val` в primary constructor без backing field?
- Почему `const val` нельзя внутри обычного класса?
