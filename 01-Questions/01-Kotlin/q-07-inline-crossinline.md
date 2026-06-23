---
type: question
category: kotlin
difficulty: senior
tags: [kotlin, inline, performance]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# `inline`, `noinline`, `crossinline`, `reified` — что это и зачем?

## Краткий ответ (TL;DR)

`inline` подставляет **тело функции** в место вызова → лямбды не аллоцируются. `noinline` отключает инлайнинг для конкретной лямбды (чтобы её можно было передать дальше или сохранить). `crossinline` — лямбда инлайнится, но в ней запрещён non-local return. `reified` — сохраняет тип-параметр в рантайме.

## Развёрнутый ответ

### Зачем inline

Обычная функция с lambda-параметром компилируется в `Function0/1/...` объект → аллокация на каждый вызов. `inline` убирает её:

```kotlin
inline fun measure(block: () -> Unit) {
    val start = System.nanoTime()
    block()
    println("took ${System.nanoTime() - start}")
}

measure { doWork() }
// в bytecode:
val start = System.nanoTime()
doWork()
println("took ${System.nanoTime() - start}")
```

**Бонус:** в инлайн-лямбде можно делать **non-local return** — `return` выйдет из вызывающей функции, а не только из лямбды.

### noinline

Если у тебя несколько лямбд и одну из них надо передать дальше / сохранить:

```kotlin
inline fun foo(a: () -> Unit, noinline b: () -> Unit) {
    a()
    saveCallback(b)  // нужна реальная Function0 — иначе нечего сохранять
}
```

### crossinline

Запрещает non-local return — нужно, когда лямбду нельзя выполнить в контексте вызова напрямую (например, она запускается в другом потоке/обработчике):

```kotlin
inline fun runOnUi(crossinline block: () -> Unit) {
    handler.post { block() }    // здесь return из block уронил бы программу
}
```

### reified

```kotlin
inline fun <reified T> Gson.fromJson(json: String): T =
    fromJson(json, T::class.java)

val user: User = gson.fromJson(jsonStr)
```

Только в `inline` функциях — компилятор подставляет реальный тип.

### Когда НЕ inline

- Большое тело → раздувает bytecode (увеличение размера APK + cache pressure).
- Нет лямбд-параметров → нет смысла.
- `inline` запрещает доступ к `private` членам класса из call site.

### Правило большого пальца

Инлайнить функции **с лямбдами**, которые часто вызываются в hot path. Не инлайнить ради «оптимизации» обычные функции.

## Подводные камни

- `inline` функция и её `inline`-лямбды не могут вызывать `private` функции класса (call site может быть в другом модуле).
- Большой `inline` → линкер таскает за собой всё, что в нём → APK раздувается.
- `inline class` — это **устаревшее имя** для `value class`. Не путать с `inline fun`.

## Связанные темы

- [[q-05-generics-variance]]
- [[q-08-sequences-vs-collections]]

## Follow-up

- Что такое non-local return и почему он опасен в `crossinline`?
- Когда `inline` ухудшает производительность?
