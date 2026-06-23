---
type: question
category: kotlin
difficulty: middle
tags: [kotlin, functional]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Функциональные типы, лямбды, function references, higher-order functions

## Краткий ответ (TL;DR)

В Kotlin функция — first-class: `(Int) -> String` это полноценный тип. Лямбды — объекты типа `Function0/1/...` (если не `inline`). Function references: `::topLevel`, `obj::method`, `Class::member`. Higher-order — функции, принимающие/возвращающие функции.

## Развёрнутый ответ

### Функциональный тип

```kotlin
val transform: (Int) -> String = { i -> i.toString() }
val checker: Int.() -> Boolean = { this > 0 }    // с receiver
val nullable: ((String) -> Int)? = null
val suspending: suspend () -> Int = { delay(1); 42 }
```

Каждый компилируется в:
- Обычная — `Function0<Int>`, `Function1<Int, String>`, ...
- С receiver — то же, но первый параметр = receiver.
- Suspend — добавляется параметр `Continuation`.

### Function references

```kotlin
fun isOdd(x: Int) = x % 2 == 1
val pred: (Int) -> Boolean = ::isOdd

class P { fun f(x: Int) = x * 2 }
val p = P()
val bound = p::f                // bound — захватывает receiver
val unbound: (P, Int) -> Int = P::f   // unbound — receiver в параметрах
```

`::isOdd` создаёт объект-обёртку (Function1). Внутри `inline` функций может оптимизироваться.

### Higher-order functions

```kotlin
fun <T, R> List<T>.fold(init: R, op: (R, T) -> R): R {
    var acc = init
    for (e in this) acc = op(acc, e)
    return acc
}
```

С `inline` лямбда «вставляется» — никаких аллокаций.

### Closures

```kotlin
fun counter(): () -> Int {
    var count = 0
    return { ++count }    // захватывает count "по ссылке" (Ref<Int>)
}
```

В Java лямбда захватывает только effectively final. Kotlin позволяет mutable, оборачивая в `Ref` (= `IntRef`/`ObjectRef`).

### Trailing lambda

```kotlin
list.filter { it > 0 }                // лямбда — единственный аргумент
list.foldRight(0) { e, acc -> e + acc }   // вне скобок, если последний
```

### Function как property delegate

```kotlin
class P {
    val f: () -> Int = { compute() }   // хранится как field типа Function0
}
```

Если property имеет лямбду — аллокация Function0 при создании объекта. Для hot path лучше `inline fun` или метод.

### SAM конверсии

```kotlin
// Java interface
interface OnClickListener { void onClick(View v); }

// Kotlin:
button.setOnClickListener { v -> handle(v) }
```

С Kotlin 1.4+ есть `fun interface`:

```kotlin
fun interface Validator<T> { fun validate(x: T): Boolean }
val v = Validator<String> { it.isNotBlank() }
```

## Подводные камни

- Лямбды — объекты. В hot path без `inline` это GC pressure.
- Method reference `obj::method` создаёт **новый** объект каждый раз — не используй как ключ в map.
- Suspend-функцию нельзя передать как обычную `(Int) -> Int` — типы разные.
- В Kotlin 2.x: SAM-конверсия для Kotlin-интерфейсов работает только через `fun interface`.

## Связанные темы

- [[q-07-inline-crossinline]]
- [[q-09-extensions]]

## Follow-up

- Что такое `fun interface` и чем отличается от обычного interface?
- Сколько аллокаций в `list.map { it * 2 }.filter { it > 0 }`?
