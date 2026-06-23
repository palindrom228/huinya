---
type: question
category: ui
difficulty: senior
tags: [compose, performance, recomposition]
created: 2026-06-16
last_reviewed: 2026-06-16
times_failed: 0
times_passed: 1
status: learning
---

# Compose performance: skippable, stable, deferred reads, baseline profiles

## Краткий ответ (TL;DR)

Главные рычаги: (1) делать composables **skippable** (стабильные параметры) → меньше рекомпозиций; (2) **отложенные чтения** state (lambda Modifier, `derivedStateOf`) → переводить работу с recomposition на layout/draw; (3) `key()` в LazyList для стабильной идентичности; (4) **Compose Compiler Metrics** для аудита; (5) **Baseline Profiles** для cold start / scroll.

## Развёрнутый ответ

### Skippable / restartable

Compose может пропустить вызов composable если **все параметры стабильны и `equals` старым**. Параметр стабилен если:

- Примитив или `String`.
- Аннотирован `@Stable` / `@Immutable`.
- Функциональный тип (но lambdas, захватывающие mutable, считаются нестабильными).
- `data class` со всеми стабильными полями (без mutable коллекций).

**Нестабильные по умолчанию**: `List`, `Map`, `Set` (интерфейсы — могут быть mutable), generic `T` без bound.

### Compose Compiler Metrics

```gradle
kotlin {
    composeCompiler {
        reportsDestination = layout.buildDirectory.dir("compose_reports")
        metricsDestination = layout.buildDirectory.dir("compose_metrics")
    }
}
```

Отчёты:
- `app_release-composables.txt` — restartable/skippable статус каждого composable.
- `app_release-classes.txt` — stability классов (`stable`, `unstable`, `runtime`).

```
restartable skippable scheme("[androidx.compose.ui.UiComposable]") fun ProductCard(
    stable product: Product
    unstable onClick: Function0<Unit>     ← НЕ stable lambda
)
```

### Deferred reads

```kotlin
// плохо: чтение в composition → рекомпозиция на каждый scroll
Box(Modifier.offset(x = scroll.value.dp))

// хорошо: чтение в layout-lambda → только layout
Box(Modifier.offset { IntOffset(scroll.value, 0) })

// хорошо: чтение в draw → только redraw
Canvas(Modifier) { drawCircle(..., radius = anim.value) }
```

### LazyColumn keys

```kotlin
LazyColumn {
    items(list, key = { it.id }) { item ->
        Row(item)
    }
}
```

Без `key` reordering вызывает recomposition всех видимых. С `key` Compose reuses items по ID.

`contentType` (lambda) — позволяет переиспользовать ViewHolder-подобные slots:

```kotlin
items(list, key = { it.id }, contentType = { it::class }) { ... }
```

### Стабилизация лямбд

```kotlin
// нестабильно: каждая recomposition новая lambda
Button(onClick = { vm.save() })

// стабильно: method reference
Button(onClick = vm::save)
```

Также можно использовать `remember { { ... } }`, но это runtime overhead.

### Immutable коллекции

```kotlin
// kotlinx.collections.immutable
val items: ImmutableList<Item> = persistentListOf(...)

@Composable fun List(items: ImmutableList<Item>) { ... }
```

`ImmutableList` помечен `@Immutable` → skippable.

Альтернатива: `@Immutable data class Items(val list: List<Item>)` — оборачивает обещанием неизменности.

### Baseline Profiles

См. [[../02-Android-Core/q-20-app-startup]]. Для Compose особенно важны: AOT-компиляция compose runtime даёт +30–40% к first frame.

### Layout Inspector / Recomposition counts

Android Studio → Layout Inspector → включить «Show recomposition counts». Видно сколько раз каждый composable рекомпозируется. Скиппнутый — отдельный счётчик.

### Modifier order

```kotlin
Box(Modifier.padding(16.dp).background(Red))   // padding снаружи
Box(Modifier.background(Red).padding(16.dp))   // padding внутри (фон шире)
```

Не perf-проблема, но частая ошибка.

### Recomposition scope optimization

Передавай **scope-функции** вместо state, чтобы изолировать чтение:

```kotlin
// плохо: ParentComposable читает count → ре-композит всё
@Composable fun Parent(count: Int) {
    Column { Heavy(); Counter(count) }
}

// хорошо: чтение в лямбде
@Composable fun Parent(countProvider: () -> Int) {
    Column { Heavy(); Counter(countProvider) }
}

@Composable fun Counter(provide: () -> Int) {
    Text("${provide()}")    // чтение здесь, только Text рекомпозируется
}
```

## Подводные камни

- `List<T>` параметр → всегда `unstable` → нет skipping. Решение: `ImmutableList` или `@Immutable` wrapper.
- Передача всего ViewModel в composable → нестабильный (класс ViewModel).
- Compose Compiler Metrics только в release-сборке адекватны (debug имеет лишние группы).
- Включённый `LiveLiterals` (по умолчанию debug) → лишняя оборачивание литералов → recomposition больше. Отключать в perf-тестах.
- Compose `Modifier.composed { ... }` создаёт нестабильность; использовать `Modifier.Node` API (1.6+) для кастомных модификаторов.

## Связанные темы

- [[q-01-compose-mental-model]]
- [[q-06-compose-lists]]
- [[../02-Android-Core/q-20-app-startup]]

## Follow-up

- Как сделать composable skippable, если параметр — `List`?
- Зачем читать state в lambda Modifier, а не в теле composable?
- Что показывает Compose Compiler Metrics?
