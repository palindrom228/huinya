---
type: category-index
category: performance-memory
tags: [index, performance-memory]
---

# 7. Performance & Memory

Профилирование, утечки, ANR, рендеринг, оптимизации, baseline profiles.

## Подтемы

- Android Studio Profiler: CPU, Memory, Network, Energy
- Перфокат: Systrace / Perfetto, Macrobenchmark, Microbenchmark
- ANR: причины, что попадает в trace, как ловить
- Утечки памяти: типичные кейсы (Context, Handler, listeners, статика)
- LeakCanary под капотом
- GC в ART, generational GC, large object allocations
- Bitmap-оптимизации, image loading (Glide / Coil), пулы
- Frame rate, jank, choreographer, vsync
- Compose performance: skippability, recomposition counts
- App startup: cold/warm/hot, App Startup library, baseline profiles
- APK size: R8, shrinking, resource shrinking, App Bundle
- Network performance: prefetching, batching, gzip

## Вопросы

```dataview
TABLE WITHOUT ID
  file.link AS "Вопрос",
  difficulty AS "Сложность",
  status AS "Статус",
  times_failed AS "Провалов"
FROM "01-Questions/07-Performance-Memory"
WHERE type = "question"
SORT difficulty DESC, status ASC
```
