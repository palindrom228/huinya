---
type: category-index
category: testing
tags: [index, testing]
---

# 8. Testing

Unit, integration, UI, тестирование корутин/Flow, моки, screenshot-тесты.

## Подтемы

- Test pyramid в Android
- JUnit 4 vs 5, Kotest
- MockK vs Mockito (final классы Kotlin, mockkObject, mockkStatic)
- Coroutine testing: `runTest`, `StandardTestDispatcher`, `UnconfinedTestDispatcher`, `TestScope`
- Flow testing: Turbine
- Robolectric, локальные unit-тесты с фреймворком Android
- Espresso, IdlingResource, Barista
- UI Automator
- Compose testing API
- Screenshot тесты: Paparazzi, Shot, Roborazzi
- Hilt testing, fake / mock зависимостей
- Test doubles: fake / stub / mock / spy
- TDD / BDD
- Flaky-тесты, retry-стратегии

## Вопросы

```dataview
TABLE WITHOUT ID
  file.link AS "Вопрос",
  difficulty AS "Сложность",
  status AS "Статус",
  times_failed AS "Провалов"
FROM "01-Questions/08-Testing"
WHERE type = "question"
SORT difficulty DESC, status ASC
```
