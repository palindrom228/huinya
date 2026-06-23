---
type: question
category: kotlin
difficulty: middle
tags: [kotlin, null-safety]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Как устроена null-safety в Kotlin? `?.`, `?:`, `!!`, `lateinit`, `Nothing?`

## Краткий ответ (TL;DR)

Тип на уровне системы типов делится на nullable (`T?`) и non-null (`T`). Операторы: `?.` — safe call, `?:` — elvis, `!!` — throw NPE если null, `lateinit` — non-null без инициализации (только `var`, не для примитивов), `Nothing?` — единственное значение `null`.

## Развёрнутый ответ

### Smart cast

После проверки на `null` компилятор автоматически кастует тип:

```kotlin
fun length(s: String?): Int {
    if (s == null) return 0
    return s.length // smart cast в String
}
```

Не работает, если значение может измениться между чеком и использованием — например, `var` из другого модуля, или property с custom getter.

### `lateinit` vs `by lazy`

| | `lateinit var` | `by lazy { ... }` |
|---|----------------|-------------------|
| Mutability | `var` | `val` |
| Thread-safe | Нет | Да (по умолчанию SYNCHRONIZED) |
| Примитивы | ❌ | ✅ |
| Кастомные setters | ❌ | ❌ |
| Чек инициализации | `::prop.isInitialized` | — |

### Платформенные типы

При вызове Java-методов Kotlin не знает, nullable значение или нет → платформенный тип `String!`. Компилятор разрешает любое использование, но в рантайме можно поймать NPE. Best practice: аннотировать Java-код `@Nullable`/`@NonNull` или явно типизировать.

### `!!.` — это плохо?

Не всегда. Иногда инвариант гарантирует не-null (типа `findViewById` после `setContentView` со 100% уверенным id). Но обычно `!!` — запах: подумай, можно ли `?.let { ... }` / `?: error("...")` / `requireNotNull(...)`.

## Подводные камни

- Smart cast не работает для `open val` (можно переопределить с custom getter).
- `lateinit` не для примитивов — для них `Delegates.notNull()`.
- `null` is `Nothing?` — поэтому `listOf(null)` возвращает `List<Nothing?>`.
- `?.let { }` возвращает `null`, если получатель `null` — НЕ выполняется блок.

## Код

```kotlin
val name: String? = getName()
// Idioms:
name?.let { println(it) }            // выполнить если не-null
name?.length ?: 0                    // default
requireNotNull(name) { "name required" }
checkNotNull(name)
val len = name!!.length              // throw NPE
```

## Связанные темы

- [[q-03-scope-functions]]
- [[q-15-jvm-interop]]

## Follow-up

- Чем отличается `?.let { }` от `if (x != null) { ... }`?
- Что вернёт `null is Any?`? А `null is Nothing?`?
- Почему smart cast не работает для `var`-полей из другого модуля?
