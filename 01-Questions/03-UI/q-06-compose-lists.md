---
type: question
category: ui
difficulty: middle
tags: [compose, lazy-list, performance]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# LazyColumn / LazyRow / LazyVerticalGrid: keys, contentType, paging

## Краткий ответ (TL;DR)

`LazyColumn` — Compose-аналог RecyclerView. Композирует только видимые элементы, переиспользует slots. Для стабильной анимации/перерасчётов нужны `key` (стабильный ID) и `contentType` (для reuse разнородных элементов). Большие списки — через **Paging 3** + `collectAsLazyPagingItems()`.

## Развёрнутый ответ

### Базовый синтаксис

```kotlin
LazyColumn(
    state = rememberLazyListState(),
    contentPadding = PaddingValues(16.dp),
    verticalArrangement = Arrangement.spacedBy(8.dp)
) {
    item { Header() }
    items(list, key = { it.id }, contentType = { it.type }) { item ->
        Row(item)
    }
    itemsIndexed(list) { i, item -> ... }
    stickyHeader { Header() }
}
```

### Why `key`

Без key: при удалении первого элемента Compose думает, что изменились все — recomposes всё видимое. С key: Compose видит, что у `id=42` параметры не изменились → переиспользует существующий slot и просто сдвигает позицию.

Также ключи нужны для:
- `animateItemPlacement()` — анимация перестановки.
- Состояния scroll position через rotation.

```kotlin
items(list, key = { it.id }) { item ->
    Row(Modifier.animateItemPlacement(), item)
}
```

### contentType

```kotlin
items(list, key = { it.id }, contentType = { it::class }) { ... }
```

LazyList переиспользует слот, только если новый item имеет тот же `contentType`. Если разные типы (header, content, footer) — указать contentType, иначе создание composable дороже.

### LazyListState

```kotlin
val state = rememberLazyListState()
LazyColumn(state = state) { ... }

// scroll programmatically
scope.launch { state.animateScrollToItem(10) }

// observe scroll
val firstVisible by remember { derivedStateOf { state.firstVisibleItemIndex } }
```

### Paging 3

```kotlin
@Composable
fun FeedScreen(vm: FeedVm = hiltViewModel()) {
    val items: LazyPagingItems<Post> = vm.pager.flow.collectAsLazyPagingItems()

    LazyColumn {
        items(count = items.itemCount, key = items.itemKey { it.id }) { i ->
            val post = items[i]
            if (post != null) PostRow(post) else PlaceholderRow()
        }

        when (items.loadState.append) {
            is LoadState.Loading -> item { Loader() }
            is LoadState.Error -> item { RetryRow { items.retry() } }
            else -> {}
        }
    }
}
```

### LazyVerticalGrid / LazyHorizontalGrid

```kotlin
LazyVerticalGrid(
    columns = GridCells.Adaptive(120.dp),
    contentPadding = PaddingValues(8.dp),
    horizontalArrangement = Arrangement.spacedBy(8.dp),
    verticalArrangement = Arrangement.spacedBy(8.dp)
) {
    items(photos, key = { it.id }) { PhotoCell(it) }
}
```

`GridCells.Fixed(n)` — n колонок; `Adaptive(minSize)` — заполнение по min ширине.

### LazyVerticalStaggeredGrid (1.5+)

Pinterest-like layout. `StaggeredGridCells.Adaptive(160.dp)`.

### Pre-fetching

LazyColumn автоматически prefetch'ит N элементов вперёд (`NestedPrefetchScope`). Можно повлиять через `LazyListState.prefetchScheduler` (advanced).

### Лучшие практики

- **Используй keys** для динамичных списков.
- **Не оборачивай весь LazyColumn в Column со scroll** — сломается виртуализация.
- **Не нестингуй** `LazyColumn` внутри вертикального scrollable родителя (без `Modifier.height` или специальных layout) → exception.
- **Передавай stable lambdas**: `onClick = { onClickItem(item.id) }` — каждый раз новая → ломает skipping строки. Лучше hoist в `remember(item.id)`.

## Подводные камни

- Без `key` — после удаления элемента вся видимая часть рекомпозируется + теряются ID для анимаций.
- `key` должен быть **уникальным и стабильным** (не index!).
- LazyColumn внутри vertical scroll Column → `IllegalStateException: Vertically scrollable component was measured with infinity max constraint`.
- `items(list) { ... }` принимает `List<T>` — для `Flow` сначала собирай в state.
- `LazyPagingItems.itemKey` критичен — без него Paging recomposes часто при `invalidate`.

## Связанные темы

- [[q-05-compose-performance]]
- [[q-17-recyclerview]]
- [[../06-Data-Networking/q-paging]]

## Follow-up

- Зачем `key` в LazyColumn?
- Когда указывать `contentType` и что это даёт?
- Почему нельзя положить `LazyColumn` в `Column.verticalScroll`?
