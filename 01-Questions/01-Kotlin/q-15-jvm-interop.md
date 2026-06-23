---
type: question
category: kotlin
difficulty: senior
tags: [kotlin, jvm, interop]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Java-interop: `@JvmStatic`, `@JvmField`, `@JvmOverloads`, `@JvmName`, default args, companion

## Краткий ответ (TL;DR)

Kotlin компилируется в JVM bytecode, но генерирует свои конвенции, которые Java видит «уродливо». Аннотации `@Jvm*` подстраивают bytecode для удобного Java-доступа.

## Развёрнутый ответ

### companion object

Из Kotlin:

```kotlin
class Foo { companion object { fun bar() = 42 } }
Foo.bar()
```

Из Java:

```java
Foo.Companion.bar();   // некрасиво
```

С `@JvmStatic`:

```kotlin
class Foo { companion object { @JvmStatic fun bar() = 42 } }
```

```java
Foo.bar();   // ✅
```

### @JvmField

Property без `@JvmField` генерирует приватное поле + getter (+ setter если var):

```kotlin
class Foo { val x = 1 }
```

Из Java: `foo.getX()`. С `@JvmField val x = 1` → доступ как `foo.x` (поле public final). Не работает с `private`, `open`, `override`, `const`.

В companion `@JvmField` делает поле статикой:

```kotlin
companion object { @JvmField val PI = 3.14 }   // Foo.PI; иначе Foo.Companion.getPI()
```

`const val` уже статика-константа: `Foo.PI` доступно напрямую.

### @JvmOverloads

Default параметры в Kotlin → в Java виден **один** метод со всеми параметрами. `@JvmOverloads` генерирует перегрузки:

```kotlin
@JvmOverloads
fun greet(name: String, prefix: String = "Hello", suffix: String = "!") { }
```

Из Java: 3 перегрузки доступны.

⚠️ Для конструкторов — `@JvmOverloads constructor(...)`.

### @JvmName

Top-level функции живут в файле `MyFileKt`:

```kotlin
// File: Utils.kt
fun foo() {}
```

Из Java: `UtilsKt.foo()`. Меняем:

```kotlin
@file:JvmName("Utils")
package x
fun foo() {}
```

`UtilsKt` → `Utils`.

Также `@JvmName` нужен при коллизиях после type erasure:

```kotlin
fun List<Int>.sum(): Int = ...
@JvmName("sumLong")
fun List<Long>.sum(): Long = ...
```

### Platform types

Java-методы без аннотаций → Kotlin видит результат как `String!` (платформенный тип). Можно использовать как nullable/non-null, но в рантайме можно поймать NPE.

Решения:
1. На Java стороне — `@Nullable`/`@NonNull` (jakarta/jsr305/androidx).
2. Включить strict mode: `-Xjsr305=strict`.
3. На Kotlin стороне — явно типизировать как `String?`.

### Checked exceptions

В Kotlin их нет → Java думает, что метод не бросает. Если нужно — `@Throws`:

```kotlin
@Throws(IOException::class)
fun read() { ... }
```

### Property in interface

В Java interface нет полей. Kotlin `interface { val x: Int }` генерирует абстрактный `getX()`. `@JvmField` в интерфейсе **нельзя** — нет реализации.

## Подводные камни

- `companion object` без `@JvmStatic`/`@JvmField` → `Foo.Companion.x` из Java.
- `const val` ставит значение **inline** в место использования → бинарная несовместимость при изменении.
- Default args в Kotlin без `@JvmOverloads` дают **один** Java-метод с лишним int-маской.
- `@JvmName` на open/override методах — нельзя.
- Property с custom getter/setter не может иметь `@JvmField`.

## Связанные темы

- [[q-01-val-vs-var-vs-const]]
- [[q-02-null-safety]]

## Follow-up

- Как из Java вызвать suspend-функцию?
- Чем `@JvmStatic` отличается от `@JvmField` в companion?
