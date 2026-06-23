---
type: question
category: kotlin
difficulty: senior
tags: [kotlin, generics, variance]
created: 2026-06-16
last_reviewed: 2026-06-16
times_failed: 1
times_passed: 0
status: learning
---

# Generics и variance: `in`, `out`, `*`, reified — что значит и когда применять?

## Краткий ответ (TL;DR)

- `out T` — **covariance**, продюсер: тип только возвращается. `List<out T>`.
- `in T` — **contravariance**, консьюмер: тип только принимается. `Comparator<in T>`.
- `*` — **star projection**: «какой-то конкретный тип, мне всё равно»; читать можно как `Any?`, записывать — нельзя.
- `reified` — сохраняет тип-параметр в рантайме (только для `inline` функций).

## Развёрнутый ответ

### Почему дженерики инвариантны по умолчанию

`MutableList<String>` НЕ является `MutableList<Any>`, иначе:

```kotlin
val s: MutableList<String> = mutableListOf("a")
val a: MutableList<Any> = s   // если бы это разрешили...
a.add(1)                      // ...мы бы положили Int в список строк
```

### Declaration-site variance (на классе)

```kotlin
interface Source<out T> { fun get(): T }            // только возвращает
interface Sink<in T>    { fun put(item: T) }        // только принимает
```

Теперь `Source<String>` это `Source<Any>` (covariant), а `Sink<Any>` это `Sink<String>` (contravariant).

### Use-site variance (на месте применения)

```kotlin
fun copy(from: Array<out Any>, to: Array<Any>) { ... }
```

Эквивалент Java `? extends`/`? super`.

### Star projection

```kotlin
fun printAll(list: List<*>) {
    list.forEach { println(it) }  // it: Any?
}
```

Полезно, когда тип параметра неизвестен и неважен. Запись/добавление чего-то конкретного запрещено.

### reified

```kotlin
inline fun <reified T> Bundle.getParcel(key: String): T? =
    if (Build.VERSION.SDK_INT >= 33) getParcelable(key, T::class.java)
    else @Suppress("DEPRECATION") getParcelable(key) as? T

inline fun <reified T : Activity> Context.start() {
    startActivity(Intent(this, T::class.java))
}
```

Под капотом `inline` подставляет тело в место вызова → компилятор знает реальный тип и вставляет его вместо `T::class`.

## Подводные камни

- `out T` запрещает `T` в позиции параметра — иначе сломается типизация.
- `reified` обязательно требует `inline` функцию.
- Type erasure в JVM остаётся: рантайм всё равно не знает `List<String>` vs `List<Int>` без reified.
- `*` ≠ `Any?` — `MutableList<*>` нельзя добавлять `Any?`, можно читать.

## Мнемоника

**PECS** из Java: **P**roducer `extends` (= `out`), **C**onsumer `super` (= `in`).

## Связанные темы

- [[q-07-inline-crossinline]]
- [[q-11-kotlin-reflection]]

## Follow-up

- Почему `List<T>` ковариантен, а `MutableList<T>` нет?
- Может ли `out T` появляться в `private` функции класса как параметр?
