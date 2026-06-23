---
type: section-index
tags: [mock-interview]
---

# 🎤 Mock-интервью

Полноценные прогоны интервью (technical / system-design / behavioral). Раз в 1–2 недели.

## Форматы

- **Technical screen** — 45–60 мин, 4–6 вопросов из разных категорий + 1 coding-задача
- **System design** — 60 мин, один кейс, глубокий разбор по чеклисту
- **Behavioral** — 45 мин, 5–7 STAR-историй
- **Полный loop** — все три, друг за другом

## Как провести

1. Найди интервьюера: ChatGPT в режиме интервьюера, друг, сервис (interviewing.io, Pramp).
2. Создай файл `YYYY-MM-DD-mock.md`, вставь [[Template-Mock-Interview]].
3. После: честная самооценка, action items, добавь слабые темы в [[../04-Weak-Spots/README]].

## Все mock'и

```dataview
TABLE WITHOUT ID
  file.link AS "Интервью",
  date AS "Дата",
  interviewer AS "Кто",
  company_target AS "Цель",
  verdict AS "Вердикт"
FROM "03-Mock-Interviews"
WHERE type = "mock-interview"
SORT date DESC
```
