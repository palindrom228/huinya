---
type: section-index
tags: [weak-spots]
---

# 🔥 Слабые места

Автоматически агрегируется из вопросов с `times_failed > 0` и низкими оценками на тестах.

## Топ-20 проваленных вопросов

```dataview
TABLE WITHOUT ID
  file.link AS "Вопрос",
  category AS "Категория",
  times_failed AS "Провалов",
  times_passed AS "Успехов",
  last_reviewed AS "Последний раз"
FROM "01-Questions"
WHERE type = "question" AND times_failed > 0
SORT times_failed DESC, last_reviewed ASC
LIMIT 20
```

## Категории, где больше всего провалов

```dataview
TABLE WITHOUT ID
  category AS "Категория",
  sum(rows.times_failed) AS "Всего провалов",
  length(filter(rows, (r) => r.times_failed > 0)) AS "Слабых вопросов"
FROM "01-Questions"
WHERE type = "question"
GROUP BY category
SORT sum(rows.times_failed) DESC
```

## Карточки для повторения

```dataview
TABLE WITHOUT ID
  file.link AS "SR-карточка",
  next_review AS "Срок",
  interval_days AS "Интервал",
  ease AS "Ease"
FROM "05-Spaced-Repetition"
WHERE type = "sr-card" AND status != "mastered"
SORT next_review ASC
```

## Ручные заметки

> Сюда можно вписать темы/паттерны/типичные ошибки, которые повторяются — не привязанные к одному вопросу.

- 
