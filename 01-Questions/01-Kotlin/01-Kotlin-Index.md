---
type: category-index
category: kotlin
tags: [index, kotlin]
---

# 1. Kotlin

Язык, особенности, generics, делегаты, scope-функции, корутины, Flow.

## Подтемы

- Базовый синтаксис, null-safety, smart-casts
- `data`/`sealed`/`value` классы, `object`, companion
- Generics, variance (`in`/`out`), reified, star-projection
- Делегаты (`by lazy`, `observable`, `Delegates.notNull`, кастомные)
- Scope-функции (`let`/`run`/`with`/`apply`/`also`)
- Coroutines: `suspend`, `CoroutineScope`, `Job`, `Dispatchers`, structured concurrency
- Flow: cold/hot, `StateFlow`/`SharedFlow`, операторы, backpressure
- Inline / crossinline / noinline
- Sequences vs Collections
- Kotlin Multiplatform (опционально)

## Вопросы

```dataview
TABLE WITHOUT ID
  file.link AS "Вопрос",
  difficulty AS "Сложность",
  status AS "Статус",
  times_failed AS "Провалов"
FROM "01-Questions/01-Kotlin"
WHERE type = "question"
SORT difficulty DESC, status ASC
```
