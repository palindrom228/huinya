---
type: question
category: ui
difficulty: senior
tags: [compose, recomposition, mental-model]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Jetpack Compose mental model: composable, recomposition, snapshot system

## Краткий ответ (TL;DR)

Composable-функции — это **описание UI как функции от состояния**. Compose runtime строит дерево «slots», запоминает позиции вызовов и **точечно ре-вызывает** только те composables, чьё прочитанное `State` изменилось. State хранится в **snapshot system** (MVCC-подобной), что даёт thread-safe чтение/запись без блокировок.

## Развёрнутый ответ

### Что такое @Composable

Это не обычная функция. Компилятор-плагин добавляет скрытые параметры (`Composer`, `changed`) и оборачивает тело в логику, которая:

1. Регистрирует позицию вызова (gappy buffer / slot table).
2. Запоминает прочитанные `State<T>` через `Snapshot.current.readObserver`.
3. При изменении state — помечает группу как invalid и шедулит рекомпозицию **только этой группы**.

### Recomposition

Compose НЕ перерисовывает весь экран — он переходит вниз по дереву и пропускает composables, чьи параметры `equals` старым (skippable) и которые сами не читают изменившийся state.

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Column {
        Text("Count: $count")          // ← рекомпозируется
        Button(onClick = { count++ }) { // ← не рекомпозируется (lambda стабильная)
            Text("Inc")
        }
    }
}
```

### Skippable / Restartable

- **Restartable**: функция, которую runtime может вызвать заново (не inline, не возвращает значение).
- **Skippable**: все параметры **стабильны** (`@Stable`/`@Immutable`/примитивы/lambdas без захвата изменяемого состояния).

Если параметр нестабилен — Compose не может сравнить → всегда рекомпозирует. Проверить: `./gradlew :app:assembleRelease -P kotlin.compose.compiler.metrics=...` и читать `composables.txt`.

### Стабильность

```kotlin
@Immutable
data class User(val id: Long, val name: String)   // обещание: не меняется

@Stable
interface Repo { val flow: Flow<List<User>> }     // меняется, но изменения наблюдаемы
```

Без аннотации `data class` со всеми `val` и стабильными типами → стабилен автоматически. С `List<User>` (mutable интерфейс) → **нестабилен** (хотя реально immutable). Решение: `ImmutableList` (kotlinx.collections.immutable) или `@Immutable`.

### Snapshot System

Compose использует MVCC-подобную модель:

- Каждый `mutableStateOf` — это snapshot-aware объект.
- Чтение запоминается в `Snapshot.current`.
- Запись создаёт новую версию; при `Snapshot.takeMutableSnapshot { ... }.apply()` изменения видны.
- Все state-чтения автоматически отслеживаются → автоматическая инвалидация.

### Три фазы

1. **Composition** — выполнение composable, генерация дерева LayoutNode.
2. **Layout** — measure + place (можно читать state — будет re-layout без recomposition).
3. **Draw** — отрисовка (можно читать state — будет re-draw без recomposition/layout).

Поэтому `Modifier.offset { IntOffset(scroll.value, 0) }` (lambda) **быстрее**, чем `Modifier.offset(x = scroll.value.dp)` — первый перетягивает только layout, второй — recomposition + layout + draw.

### Positional memoization

`remember { ... }` запоминает значение **по позиции вызова в slot table**. Поэтому:

```kotlin
if (flag) {
    val a = remember { Heavy() }   // одна позиция
} else {
    val a = remember { Heavy() }   // ДРУГАЯ позиция, новый Heavy
}
```

И поэтому нельзя вызывать composables в обычном `if/while` с движущимися условиями без `key()`.

## Подводные камни

- `List<T>`, `Map`, `Set` — **нестабильны** (mutable интерфейсы) → ломают skipping.
- Лямбда, захватывающая изменяемую переменную → нестабильная (новый instance при каждой рекомпозиции).
- Long-running работа в `@Composable` (без `LaunchedEffect`) → выполнится при каждой recomposition.
- `mutableStateOf(SomeClass())` без `remember` → state создаётся заново на каждой recomposition (баг).
- Чтение `State` в lambda Modifier — не triggers recomposition (это feature!), но новички ждут реакции.

## Связанные темы

- [[q-02-compose-state]]
- [[q-03-compose-side-effects]]
- [[q-05-compose-performance]]

## Follow-up

- Что значит «skippable» и «restartable» в Compose?
- Чем `@Stable` отличается от `@Immutable`?
- Почему `Modifier.offset { ... }` (lambda) быстрее обычного `offset(x.dp)`?
