---
type: category-index
category: architecture
tags: [index, architecture]
---

# 4. Architecture & DI

Паттерны представления, чистая архитектура, SOLID, DI, модуляризация.

## Подтемы

- MVC / MVP / MVVM / MVI — отличия, когда что
- Clean Architecture (entities / use cases / repositories / data sources)
- Unidirectional Data Flow, single source of truth
- Repository pattern, mappers, DTO/Domain/UI модели
- DI: Hilt, Dagger 2, Koin — сравнение, scope-ы, multibinding
- Service Locator vs DI
- Многомодульность: feature / core / data / domain; dynamic feature modules
- Navigation между модулями, контракты, deeplinks
- App Architecture Guidelines от Google
- SOLID на примерах в Android

## Вопросы

```dataview
TABLE WITHOUT ID
  file.link AS "Вопрос",
  difficulty AS "Сложность",
  status AS "Статус",
  times_failed AS "Провалов"
FROM "01-Questions/04-Architecture"
WHERE type = "question"
SORT difficulty DESC, status ASC
```
