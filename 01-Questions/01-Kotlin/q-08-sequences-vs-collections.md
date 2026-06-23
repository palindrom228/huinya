---
type: question
category: kotlin
difficulty: middle
tags: [kotlin, collections, performance]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Чем `Sequence` отличается от `Collection`? Когда использовать?

## Краткий ответ (TL;DR)

Операции на **коллекциях** — **eager**, **horizontal**: каждая операция строит промежуточный список. На **sequences** — **lazy**, **vertical**: элементы проходят через всю цепочку по одному, без промежуточных коллекций.

## Развёрнутый ответ

### Eager (List/Set)

```kotlin
list.map { it * 2 }      // создаёт List
    .filter { it > 10 }  // создаёт ещё один List
    .first()             // берёт первый
```

Если в `list` 1M элементов и нам нужен только первый, подходящий под условие — мы всё равно прошли всё дважды и создали два промежуточных списка.

### Lazy (Sequence)

```kotlin
list.asSequence()
    .map { it * 2 }
    .filter { it > 10 }
    .first()             // terminal — запускает pipeline
```

Sequence работает по принципу pull: terminal operation запрашивает по одному элементу через всю цепочку. Прекращается, как только нашли `first()`.

### Когда выбирать что

| Кейс | Лучше |
|------|-------|
| Маленькая коллекция (< 1000 элементов) | List — проще, меньше overhead |
| Большая коллекция + несколько lazy-операций | Sequence |
| `take(n)`, `first`, `find` — short-circuit | Sequence |
| Только одна операция (map/filter/sum) | List — выигрыш нулевой |
| Бесконечная последовательность | Только Sequence (`generateSequence`, `sequence { yield(...) }`) |
| Нужны индексы / `chunked`, `windowed` с переиспользованием | Обычно List |

### Под капотом

`Sequence` — это интерфейс с одним методом `iterator()`. Каждый `map/filter` возвращает обёртку, которая **переоборачивает iterator**. Terminal operation (`toList`, `first`, `sum`) запускает реальное движение по iterator.

### Бенчмарк-интуиция

Sequence имеет накладные расходы на каждый вызов (lambda call через iterator + state machine). На маленьких коллекциях List **быстрее**, потому что:
1. Прямой цикл оптимизируется JIT-ом.
2. Память локальна (array-backed).
3. Меньше виртуальных вызовов.

## Подводные камни

- `Sequence` без terminal operation — ничего не выполняется.
- Множественное прохождение sequence — `generateSequence` без `constrainOnce()` может пересоздать значения; `iterator-based` sequence может бросить exception на повторный обход.
- `sortedBy` на Sequence **полностью загружает в память** — теряет lazy.
- Не путай `Flow` (coroutines) и `Sequence` — концептуально похоже, но Flow асинхронный и suspend.

## Связанные темы

- [[q-07-inline-crossinline]]
- [[q-13-flow-basics]]

## Follow-up

- Почему `sequence.sortedBy { }.first()` теряет преимущество lazy?
- Чем `sequence { yield(); yield() }` под капотом похож на корутины?
