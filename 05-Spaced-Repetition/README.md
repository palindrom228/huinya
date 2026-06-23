---
type: section-index
tags: [sr]
---

# 🔁 Spaced Repetition

Интервальные повторения по упрощённому алгоритму **SM-2** (как в Anki).

## Зачем

Память забывает по экспоненте (кривая Эббингауза). Если повторять в правильные моменты — кривая выравнивается, и темы остаются «в активной памяти» к собеседованию.

## Что попадает в SR

- Любой вопрос с оценкой ≤ 1 на тестировании
- Любая тема, которую забыл на mock-интервью
- Сложные концепты, на которые сам себе указал

## Как создать карточку

1. После проваленного вопроса создай файл `<slug>.md` в этой папке.
2. Вставь [[Template-SR-Card]].
3. Заполни ссылку на исходный [[Template-Question|вопрос]] и краткий ответ для front/back.

## Алгоритм SM-2 (упрощённый)

| Качество ответа | Действие |
|-----------------|----------|
| 0–2 (провал) | `repetitions = 0`, `interval = 1 день`, ease не меняется |
| 3 | `interval = old × ease`, `ease -= 0.14` (мин. 1.3) |
| 4 | `interval = old × ease` |
| 5 | `interval = old × ease`, `ease += 0.10` |

Стартовая последовательность интервалов (при качестве 4+):

`1 → 6 → 15 → 38 → 95 → 240` дней (ease=2.5)

## Ежедневный ритуал

1. Открой [[../00-Home/Dashboard|Dashboard]] — секция «Карточки на сегодня».
2. По каждой:
   - Прочти front
   - Вспомни ответ (вслух)
   - Открой back, оцени 0–5
   - Обнови `last_reviewed`, `next_review`, `interval_days`, `ease`, `repetitions`
3. Если карточка прошла 5+ успешных циклов и интервал > 90 дней → `status: mastered`.

## Очередь на сегодня

```dataview
TABLE WITHOUT ID
  file.link AS "Карточка",
  question AS "→ Вопрос",
  interval_days AS "Интервал",
  repetitions AS "Повторов"
FROM "05-Spaced-Repetition"
WHERE type = "sr-card" AND next_review <= date(today) AND status != "mastered"
SORT next_review ASC
```

## Скоро (следующие 7 дней)

```dataview
TABLE WITHOUT ID
  file.link AS "Карточка",
  next_review AS "Дата",
  interval_days AS "Интервал"
FROM "05-Spaced-Repetition"
WHERE type = "sr-card" AND next_review > date(today) AND next_review <= date(today) + dur(7 days)
SORT next_review ASC
```

## Все карточки

```dataview
TABLE WITHOUT ID
  file.link AS "Карточка",
  status AS "Статус",
  repetitions AS "Повторов",
  interval_days AS "Интервал",
  next_review AS "Следующее"
FROM "05-Spaced-Repetition"
WHERE type = "sr-card"
SORT status ASC, next_review ASC
```
