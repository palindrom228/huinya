---
type: question
category: ui
difficulty: senior
tags: [recyclerview, diff-util, adapter]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# RecyclerView: ViewHolder, Adapter, DiffUtil, prefetch, payloads, nested scrolling

## Краткий ответ (TL;DR)

RecyclerView — пул `ViewHolder` + LayoutManager + Adapter. Чтобы апдейты были эффективны: `ListAdapter` + `DiffUtil` для diff, `payload`-объекты для частичного перерисовывания одного ItemViewHolder, стабильные `itemId` (`setHasStableIds(true)`), `RecyclerView.RecycledViewPool` для шары между списками, `setItemViewCacheSize` для свежих скроллов.

## Развёрнутый ответ

### Архитектура

```
RecyclerView ─ LayoutManager  (Linear / Grid / Staggered)
             ─ Adapter        (создаёт VH, биндит данные)
             ─ Recycler/Pool  (переиспользует VH)
             ─ ItemAnimator   (анимации add/remove/move/change)
             ─ ItemDecoration (дивайдеры, паддинги)
```

### Базовый Adapter

```kotlin
class UserAdapter : ListAdapter<User, UserVh>(DIFF) {
    override fun onCreateViewHolder(p: ViewGroup, vt: Int) =
        UserVh(ItemUserBinding.inflate(p.context.inflater, p, false))
    override fun onBindViewHolder(h: UserVh, pos: Int) = h.bind(getItem(pos))

    companion object {
        val DIFF = object : DiffUtil.ItemCallback<User>() {
            override fun areItemsTheSame(o: User, n: User) = o.id == n.id
            override fun areContentsTheSame(o: User, n: User) = o == n
        }
    }
}
```

### DiffUtil

Считает minimum edit sequence двух списков (Myers' algorithm, O(N+M*D)). Запускается на background thread (`AsyncListDiffer` внутри `ListAdapter`).

- `areItemsTheSame` — это «тот же» item? (обычно `o.id == n.id`).
- `areContentsTheSame` — содержимое идентично? Если нет → `onBindViewHolder` вызовется.
- `getChangePayload(o, n)` (опц.) — что именно поменялось → bind с payload.

### Payloads

```kotlin
override fun getChangePayload(o: User, n: User): Any? {
    return if (o.name == n.name && o.avatar != n.avatar) "AVATAR" else null
}

override fun onBindViewHolder(h: UserVh, pos: Int, payloads: List<Any>) {
    if (payloads.contains("AVATAR")) h.updateAvatar(getItem(pos))
    else super.onBindViewHolder(h, pos, payloads)
}
```

Изменяешь только аватар → не перерисовываешь весь VH.

### Stable IDs

```kotlin
init { setHasStableIds(true) }
override fun getItemId(position: Int) = getItem(position).id
```

Помогает `ItemAnimator` определять перемещения; в `ConcatAdapter` помогает не конфликтовать.

### RecyclerView prefetch

`LinearLayoutManager.setItemPrefetchEnabled(true)` (default) + `setInitialPrefetchItemCount(n)` для вложенных горизонтальных списков. RecyclerView в idle frame заранее создаёт следующие VH.

### Multiple view types

```kotlin
override fun getItemViewType(pos: Int) = when (getItem(pos)) {
    is Header -> TYPE_HEADER
    is Item -> TYPE_ITEM
}

override fun onCreateViewHolder(p: ViewGroup, type: Int) = when (type) {
    TYPE_HEADER -> HeaderVh(...)
    TYPE_ITEM -> ItemVh(...)
    else -> error("unknown")
}
```

Или **ConcatAdapter** — несколько адаптеров в один список:

```kotlin
val concat = ConcatAdapter(headerAdapter, listAdapter, footerAdapter)
recyclerView.adapter = concat
```

### Shared pool

```kotlin
val pool = RecyclerView.RecycledViewPool()
outerRv.recycledViewPool = pool
innerHorizontalRv.recycledViewPool = pool   // в каждом item-VH
```

Полезно для горизонтальных «карусели» внутри вертикального списка.

### Nested scrolling

```kotlin
rv.isNestedScrollingEnabled = true
```

Координация с CoordinatorLayout / AppBarLayout (collapse). В Compose — flow `NestedScrollConnection`.

### setItemViewCacheSize

Кэш свежих detached VH без полного rebind:

```kotlin
rv.setItemViewCacheSize(20)
```

Использовать осторожно — съедает память.

### ItemDecoration

```kotlin
class DividerDecoration : RecyclerView.ItemDecoration() {
    override fun onDraw(c: Canvas, parent: RecyclerView, state: State) {
        for (i in 0 until parent.childCount) {
            val child = parent.getChildAt(i)
            c.drawLine(...)
        }
    }
    override fun getItemOffsets(o: Rect, view: View, parent: RecyclerView, s: State) {
        o.set(0, 0, 0, dividerHeight)
    }
}
```

## Подводные камни

- `notifyDataSetChanged()` ломает анимации, перерисовывает всё — использовать `DiffUtil`.
- `wrap_content` высота item-root → multiple measure passes, jank.
- Bitmap аллокация в `onBindViewHolder` → GC pressure. Кэшируй / используй Glide/Coil.
- Click listener в bind через лямбду → новый instance на каждый bind. Сохрани в VH.
- `setHasStableIds(true)` без корректного `getItemId` → анимации сломаются.

## Связанные темы

- [[q-13-views-measure-layout-draw]]
- [[q-06-compose-lists]]
- [[../07-Performance-Memory/q-jank]]

## Follow-up

- Что делает DiffUtil и почему он лучше `notifyDataSetChanged`?
- Зачем нужны `payloads` в `onBindViewHolder`?
- Когда `ConcatAdapter` лучше, чем `getItemViewType`?
