---
type: category-index
category: system-design
tags: [index, system-design]
---

# 9. System Design (Android)

Архитектура крупных приложений, офлайн, синхронизация, push, real-time.

## Типовые задачи

- Дизайн ленты (Instagram/Twitter feed)
- Дизайн мессенджера (offline + sync + push + reactions + media)
- Дизайн оффлайн-first приложения (заметки, документы)
- Загрузка/выгрузка больших файлов (chunked upload, resumable)
- Видео/аудио стриминг
- Лента с бесконечным скроллом и кэшированием
- Push-уведомления: FCM, in-app, scheduled
- Real-time: WebSocket, SSE, polling — когда что
- Multi-device sync (CRDT-подобные подходы)
- A/B тестирование и feature flags
- Аналитика, ивент-bus, batching, retry policy
- Авторизация: OAuth, refresh-token rotation, биометрия
- Карты, location-сервисы, фоновое отслеживание

## Чеклист для каждого дизайна

1. Уточнить требования (functional + non-functional)
2. Оценить масштаб (RPS, users, data, storage)
3. Высокоуровневая архитектура (модули, слои)
4. Деталь по ключевым компонентам
5. Хранение и кэширование
6. Сеть и оффлайн
7. Concurrency и threading-модель
8. Безопасность
9. Тестирование
10. Метрики, мониторинг, аналитика
11. Trade-offs и альтернативы

## Вопросы / разборы

```dataview
TABLE WITHOUT ID
  file.link AS "Кейс",
  difficulty AS "Сложность",
  status AS "Статус",
  times_failed AS "Провалов"
FROM "01-Questions/09-System-Design"
WHERE type = "question"
SORT difficulty DESC, status ASC
```
