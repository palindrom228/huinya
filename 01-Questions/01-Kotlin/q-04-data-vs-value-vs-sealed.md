---
type: question
category: kotlin
difficulty: middle
tags: [kotlin, classes]
created: 2026-06-16
last_reviewed: 2026-06-16
times_failed: 0
times_passed: 1
status: learning
---

# `data class`, `value class`, `sealed class`, `object`, `enum class` — когда что?

## Краткий ответ (TL;DR)

- **`data`** — DTO/модель: `equals/hashCode/toString/copy/componentN` бесплатно.
- **`value`** (`@JvmInline`) — type-safe wrapper над одним полем, инлайнится в bytecode → ноль аллокаций.
- **`sealed`** — закрытая иерархия для исчерпывающего `when` (ADT / state).
- **`enum`** — фиксированный набор значений, у каждого свой singleton.
- **`object`** — синглтон (или `companion object` для статики).

## Развёрнутый ответ

### data class

```kotlin
data class User(val id: Long, val name: String)
```

Под капотом — `equals` сравнивает все props в primary constructor, `copy()` создаёт shallow copy, `componentN` для destructuring (`val (id, name) = user`).

**Гонки:** props **не** в primary constructor не участвуют в `equals/hashCode`.

### value class (inline class)

```kotlin
@JvmInline
value class UserId(val raw: Long)

fun load(id: UserId) { /* type-safe, без аллокации */ }
```

Компилятор разворачивает `UserId` в `Long` в местах использования (за исключением generic-параметров и nullable). Решает проблему primitive obsession без оверхеда.

### sealed class / sealed interface

```kotlin
sealed interface UiState {
    object Loading : UiState
    data class Success(val data: List<Item>) : UiState
    data class Error(val msg: String) : UiState
}

fun render(s: UiState) = when (s) {       // exhaustive
    UiState.Loading -> showSpinner()
    is UiState.Success -> showList(s.data)
    is UiState.Error -> showError(s.msg)
}
```

Все подтипы должны быть в **той же compilation unit** (один модуль, начиная с Kotlin 1.5 — один и тот же пакет для sealed interface снято — теперь любая subclass в том же модуле). Компилятор знает все варианты → `when` без `else`.

### enum vs sealed

| | enum | sealed |
|---|------|--------|
| Кол-во экземпляров | фиксировано | разные подтипы могут иметь множество инстансов |
| State (поля) | одинаковая структура | у каждого подкласса своя |
| Используется | флаги, состояния без данных | UI state, Result, ADT |

### object

```kotlin
object Logger { fun log(msg: String) { ... } }
// companion для "статики" класса
class Repo { companion object { const val BASE = "..." } }
```

## Подводные камни

- `data class` нельзя `open` (но можно наследовать от sealed).
- `value class` не работает с `equals` через `==` для разных wrapper-ов одного типа (понятно), но имеет ограничения с `init` блоками и backing fields.
- `sealed` подклассы должны быть в одном модуле — иначе используй интерфейс + конвенцию.

## Связанные темы

- [[q-01-val-vs-var-vs-const]]
- [[q-06-delegates]]

## Follow-up

- В каких случаях `value class` боксится (превращается в реальный объект)?
- Чем `sealed interface` лучше `sealed class`?
