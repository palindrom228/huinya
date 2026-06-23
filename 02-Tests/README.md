---
type: section-index
tags: [tests]
---

# 📝 Журнал тестирований

Каждое тестирование = **12 вопросов, по 1 на категорию**. Цель — равномерное покрытие.

## Как провести

1. Создай файл `YYYY-MM-DD-test.md` в этой папке.
2. Вставь шаблон `Templates: Insert template → Template-Test` (или скопируй из [[Template-Test]]).
3. Выбери по одному вопросу из каждой категории (можно случайно: `Cmd+P → Random note` пока курсор в `01-Questions/<cat>/`).
4. Отвечай **вслух** или **письменно** в течение 2–3 минут на вопрос.
5. Проставь оценку 0–3 для каждого.
6. После теста:
   - Обнови `times_failed` / `times_passed` в каждом [[Template-Question|вопросе]]
   - Любой вопрос с оценкой ≤ 1 → добавь в [[../04-Weak-Spots/README|Weak spots]] и заведи [[Template-SR-Card|SR-карточку]]
   - Поставь `status: done` в frontmatter теста

## Все тестирования

```dataview
TABLE WITHOUT ID
  file.link AS "Тест",
  date AS "Дата",
  score AS "Баллы",
  round(score / 36 * 100) AS "%",
  duration_min AS "Мин",
  status AS "Статус"
FROM "02-Tests"
WHERE type = "test"
SORT date DESC
```

## Динамика

```dataview
TABLE WITHOUT ID
  date AS "Дата",
  score AS "Баллы /36"
FROM "02-Tests"
WHERE type = "test" AND status = "done"
SORT date ASC
```
