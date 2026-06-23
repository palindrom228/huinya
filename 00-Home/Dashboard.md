---
type: dashboard
tags: [dashboard]
---

# 🎯 Dashboard

> ⚠️ Для автоматических таблиц установи плагин **Dataview** (Settings → Community plugins → Browse → Dataview → Install & Enable).

## 📊 Общий прогресс

```dataview
TABLE WITHOUT ID
  length(rows) AS "Всего вопросов",
  length(filter(rows, (r) => r.status = "mastered")) AS "Освоено",
  length(filter(rows, (r) => r.status = "learning")) AS "В работе",
  length(filter(rows, (r) => r.status = "new")) AS "Новых"
FROM "01-Questions"
WHERE type = "question"
GROUP BY true
```

## 📈 По категориям

```dataview
TABLE WITHOUT ID
  category AS "Категория",
  length(rows) AS "Всего",
  length(filter(rows, (r) => r.status = "mastered")) AS "✅",
  length(filter(rows, (r) => r.status = "learning")) AS "🔁",
  length(filter(rows, (r) => r.status = "new")) AS "🆕"
FROM "01-Questions"
WHERE type = "question"
GROUP BY category
SORT category ASC
```

## 📝 Последние тестирования

```dataview
TABLE WITHOUT ID
  file.link AS "Тест",
  date AS "Дата",
  score AS "Баллы",
  (score / 36 * 100) AS "%",
  status AS "Статус"
FROM "02-Tests"
WHERE type = "test"
SORT date DESC
LIMIT 10
```

## 🎤 Последние Mock-интервью

```dataview
TABLE WITHOUT ID
  file.link AS "Интервью",
  date AS "Дата",
  verdict AS "Вердикт",
  company_target AS "Цель"
FROM "03-Mock-Interviews"
WHERE type = "mock-interview"
SORT date DESC
LIMIT 5
```

## 🔥 Топ слабых вопросов (проваленных)

```dataview
TABLE WITHOUT ID
  file.link AS "Вопрос",
  category AS "Категория",
  times_failed AS "Провалов",
  last_reviewed AS "Последний раз"
FROM "01-Questions"
WHERE type = "question" AND times_failed > 0
SORT times_failed DESC, last_reviewed ASC
LIMIT 15
```

## 🔁 Карточки на сегодня (SR)

```dataview
TABLE WITHOUT ID
  file.link AS "Карточка",
  next_review AS "Срок",
  interval_days AS "Интервал",
  repetitions AS "Повторов"
FROM "05-Spaced-Repetition"
WHERE type = "sr-card" AND next_review <= date(today)
SORT next_review ASC
```
