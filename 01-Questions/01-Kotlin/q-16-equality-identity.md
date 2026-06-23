---
type: question
category: kotlin
difficulty: middle
tags: [kotlin, equality]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Равенство в Kotlin: `==`, `===`, `equals`, `hashCode`. Контракт.

## Краткий ответ (TL;DR)

`==` → `equals()` (structural), `===` → reference identity. В Kotlin нет ловушки Java «забыл equals — сравнил ссылки» только если используешь `==`. Контракт `equals/hashCode`: equal-объекты обязаны иметь одинаковый `hashCode`; `hashCode` стабилен в течение жизни объекта (с константными полями).

## Развёрнутый ответ

### == vs ===

```kotlin
val a = "hello"
val b = "he" + "llo"
println(a == b)   // true  (structural)
println(a === b)  // зависит от String pool (часто true для литералов; в общем — false)
```

### data class

```kotlin
data class User(val id: Long, val name: String)
```

Авто-генерирует `equals/hashCode/toString/copy/componentN` по props в primary constructor.

**Гонка:** поля, объявленные в теле класса, **не** участвуют:

```kotlin
data class User(val id: Long) {
    var name: String = ""   // НЕ в equals/hashCode!
}
```

### Контракт equals

1. **Reflexive**: `a.equals(a) == true`.
2. **Symmetric**: `a.equals(b) == b.equals(a)`.
3. **Transitive**: `a == b && b == c ⇒ a == c`.
4. **Consistent**: повторные вызовы без изменения полей дают тот же результат.
5. `a.equals(null) == false`.

### Контракт hashCode

- `a.equals(b) ⇒ a.hashCode() == b.hashCode()`. Обратное — НЕ обязательно.
- Если объект — ключ в `HashMap`/`HashSet` и его hashCode меняется → теряется в коллекции.

### Типичная ошибка

```kotlin
class Mutable(var x: Int) {
    override fun equals(other: Any?) = (other as? Mutable)?.x == x
    override fun hashCode() = x
}
val m = Mutable(1)
val set = hashSetOf(m)
m.x = 2
set.contains(m)   // false! Объект «потерялся» в bucket.
```

### equals в иерархии

Симметричность ломается при наследовании:

```kotlin
open class Point(val x: Int, val y: Int) {
    override fun equals(other: Any?) = other is Point && x == other.x && y == other.y
}
class ColorPoint(x: Int, y: Int, val color: Color) : Point(x, y) {
    override fun equals(other: Any?) = ... // если требуем equal color → asymmetric
}
```

Стандартное решение — `getClass()` check (Java-style) или `===` на типе, либо — composition вместо наследования.

### Когда переопределять самому

- Когда `data class` не подходит (например, сравнение по бизнес-ключу, а не по всем полям).
- В sealed-иерархиях обычно достаточно `data class` для подтипов.

### Совместный override

Если переопределяешь `equals` — переопределяй `hashCode`. IDE warning + Detekt-правило.

## Подводные камни

- `==` на массивах сравнивает ссылки! Используй `contentEquals`.
- `equals` для `value class` — сравнение по полю, но при unboxing/boxing могут быть сюрпризы.
- `equals` в `Double`/`Float` — `NaN != NaN` (по JLS), но в data class это «починено» через `Double.equals(Object)`.

## Связанные темы

- [[q-04-data-vs-value-vs-sealed]]
- [[q-01-val-vs-var-vs-const]]

## Follow-up

- Что произойдёт, если в data class изменить var-поле, которое в primary constructor?
- Чем `arr1 == arr2` отличается от `arr1.contentEquals(arr2)`?
