---
type: question
category: kotlin
difficulty: middle
tags: [kotlin, extensions]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Extension-функции: как работают, ограничения, dispatch

## Краткий ответ (TL;DR)

Extension-функции — **синтаксический сахар** над статическими функциями с первым параметром-receiver. Компилируются в `static` методы, диспатч **по компайл-тайм типу** (а не runtime), не имеют доступа к `private` членам класса, не могут быть `override`.

## Развёрнутый ответ

### Под капотом

```kotlin
fun String.lastChar(): Char = this[length - 1]
```

Компилируется примерно в:

```java
public static char lastChar(String $this) {
    return $this.charAt($this.length() - 1);
}
```

Никакой реальной модификации класса `String` не происходит — это просто статика, к которой компилятор подбирает синтаксис вызова.

### Static dispatch (важно!)

```kotlin
open class Animal
class Dog : Animal()

fun Animal.name() = "animal"
fun Dog.name() = "dog"

val a: Animal = Dog()
println(a.name()) // "animal" — по типу переменной, не объекта!
```

Это часто ловит. Если нужна полиморфия — используй виртуальный метод в классе.

### Extension property

```kotlin
val String.lastIndex: Int get() = length - 1
```

Backing field у extension property **нет** — только getter (+ setter, если `var`).

### Extension с receiver двух типов

```kotlin
fun StringBuilder.appendIf(condition: Boolean, value: String) {
    if (condition) append(value)
}
```

Лямбда с receiver — основа DSL:

```kotlin
fun html(block: HtmlBuilder.() -> Unit): String { ... }
html { body { p("hi") } }
```

### Member vs Extension

Если member и extension имеют одинаковую сигнатуру — **member побеждает** при разрешении.

### Ограничения

- Нет доступа к `private`/`protected` членам класса.
- Не могут быть `open`/`override`.
- Не отображаются в Java через рефлексию класса — нужен `KtClass.kt` файл.
- Не существуют для primitive types в JVM (но компилятор делает вид).

## Подводные камни

- Static dispatch → переопределить нельзя, легко получить «не ту» функцию.
- Extension на `Any?` (nullable receiver) полезен (`x?.let { }` это пример), но злоупотребление мешает читаемости.
- Видимость — extension следует обычным правилам (`internal`, `private`), но `private` extension доступен только в файле.
- Утечка: extension с лямбдой, которая держит Context — может пережить владельца.

## Связанные темы

- [[q-03-scope-functions]]
- [[q-07-inline-crossinline]]

## Follow-up

- Почему extension-функция не может быть `override`?
- Где живут extension-функции в bytecode и как их вызывать из Java?
