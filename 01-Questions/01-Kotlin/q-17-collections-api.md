---
type: question
category: kotlin
difficulty: middle
tags: [kotlin, collections]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Коллекции в Kotlin: read-only vs immutable, `List` vs `MutableList`, конверсии

## Краткий ответ (TL;DR)

В Kotlin **нет настоящих immutable коллекций** (на стороне stdlib). `List`/`Set`/`Map` — **read-only views**, под которыми может быть mutable-коллекция. Реальная иммутабельность достигается через `kotlinx.collections.immutable` или конвертацию в `Array`/`unmodifiableList`.

## Развёрнутый ответ

### Иерархия

```
Iterable
  └─ Collection (read-only)
       ├─ List
       │    └─ MutableList
       └─ Set
            └─ MutableSet
Map (read-only)
  └─ MutableMap
```

`MutableList` наследуется от `List` — `cast` mutable → read-only бесплатный (это та же ссылка).

### Опасность read-only

```kotlin
val mutable = mutableListOf(1, 2, 3)
val readOnly: List<Int> = mutable
mutable.add(4)
println(readOnly)   // [1, 2, 3, 4] — изменилось «под рукой»
```

`List` гарантирует только что **через эту ссылку** ты не можешь модифицировать. Гарантии иммутабельности нет.

### Реальная иммутабельность

1. **`listOf(...)`** возвращает реализацию (`Arrays.ArrayList` для одиночного, `ArrayList` копию) — но статический тип `List`. Для гарантии — `Collections.unmodifiableList(...)`.
2. **`kotlinx.collections.immutable`**: `persistentListOf(1, 2, 3)` — настоящий persistent data structure с эффективным `add`.
3. **`toList()`** на mutable — копия.

### Сравнение основных билдеров

```kotlin
listOf(1, 2, 3)            // List<Int> immutable view, под капотом ArrayList
mutableListOf(1, 2, 3)     // ArrayList
arrayListOf(1, 2, 3)       // ArrayList (явно)
listOfNotNull(1, null, 3)  // [1, 3]
List(5) { it * 2 }         // [0, 2, 4, 6, 8]
buildList { add(1); add(2) }  // build с DSL
```

### Map / Set

```kotlin
mapOf("a" to 1, "b" to 2)   // LinkedHashMap (сохраняет порядок вставки)
hashMapOf(...)               // HashMap
linkedMapOf(...)             // LinkedHashMap
sortedMapOf(...)             // TreeMap
```

### Полезные операторы

```kotlin
val pairs = list.zipWithNext()                // [a,b,c] → [(a,b),(b,c)]
val grouped = users.groupBy { it.dept }       // Map<Dept, List<User>>
val partitioned = items.partition { it.x > 0 } // Pair<List, List>
val windowed = list.windowed(3, step = 1)
val chunked = list.chunked(100)
val flat = nested.flatten()                    // List<List<T>> → List<T>
val assoc = users.associate { it.id to it }    // Map<Id, User>
val associateBy = users.associateBy { it.id }  // тоже Map<Id, User>
```

### Mutable vs immutable performance

- `toList()` — копия O(n).
- `+` для List создаёт новый список (O(n)).
- `mutableList += x` — O(1) амортизировано.
- `persistentList.add(x)` — O(log32 n) благодаря trie.

## Подводные камни

- «Read-only ≠ immutable» — основная ловушка.
- `mapOf("a" to 1)` создаёт `Map<String, Int>` — порядок сохраняется (LinkedHashMap), но в read-only типе это не гарантировано контрактом.
- `Iterator` потребляется один раз — повторный обход sequence/iterable из лямбды может удивить.
- `Map.getOrDefault` (Java) vs `getOrDefault` (Kotlin) — есть на JDK 8+, проверяй minSdk.

## Связанные темы

- [[q-08-sequences-vs-collections]]
- [[q-16-equality-identity]]

## Follow-up

- Почему `List<Int>` ковариантен, а `MutableList<Int>` — нет?
- Когда выбирать `persistentListOf` vs `listOf`?
