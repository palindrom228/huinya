---
type: question
category: kotlin
difficulty: senior
tags: [kotlin, delegates]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Делегаты в Kotlin: `by lazy`, `Delegates.observable`, делегирование классов, кастомные делегаты

## Краткий ответ (TL;DR)

Делегат — объект, в который компилятор перенаправляет вызовы `get`/`set` свойства или методы интерфейса. Бывают:
1. **Property delegates** — `by`, реализуют `getValue`/`setValue` (или `ReadOnlyProperty`/`ReadWriteProperty`).
2. **Class delegates** — `class A(b: B) : I by b` — все методы `I` перенаправляются в `b`.

## Развёрнутый ответ

### Property delegate

```kotlin
class User(map: Map<String, Any?>) {
    val name: String by map         // делегирует чтение в map["name"]
}

val name: String by lazy { computeExpensive() }   // первый раз — вычисляет, дальше кеш
var x: Int by Delegates.observable(0) { _, old, new -> log("$old -> $new") }
var nonNull: String by Delegates.notNull()
```

### Кастомный делегат

```kotlin
class Pref<T>(
    private val key: String,
    private val default: T,
    private val prefs: SharedPreferences
) : ReadWriteProperty<Any?, T> {
    override fun getValue(thisRef: Any?, property: KProperty<*>): T = TODO()
    override fun setValue(thisRef: Any?, property: KProperty<*>, value: T) = TODO()
}

class Settings(prefs: SharedPreferences) {
    var token: String by Pref("token", "", prefs)
}
```

С `operator fun getValue/setValue` можно вообще без интерфейсов — структурная типизация.

### Class delegation

```kotlin
interface Repo { fun load(): List<Item> }
class CachedRepo(private val origin: Repo, private val cache: Cache) : Repo by origin {
    override fun load(): List<Item> = cache.get() ?: origin.load().also(cache::put)
}
```

Composition over inheritance: переопределяем только нужное, остальное проксируется.

### `by lazy` режимы

```kotlin
val a by lazy(LazyThreadSafetyMode.SYNCHRONIZED) { ... }  // default, double-check lock
val b by lazy(LazyThreadSafetyMode.PUBLICATION) { ... }   // могут вычислить несколько раз, побеждает первый
val c by lazy(LazyThreadSafetyMode.NONE) { ... }          // без синхронизации, только если уверен в single-thread
```

### Под капотом

Компилятор для `val x by D()` генерирует:
- `private val x$delegate = D()`
- `fun getX() = x$delegate.getValue(this, ::x)`

В нём `::x` — `KProperty` метаданные → можно использовать имя/тип свойства внутри делегата (например, ключ в `Map`/`SharedPreferences`).

## Подводные камни

- `lateinit` ≠ `lazy`: `lateinit var`, без блока инициализации.
- `by lazy` для `Context`-зависимых полей в Activity — может держать ссылку через лямбду → утечка, если объект переживает Activity.
- Class delegation НЕ копирует методы — вызовы реально идут через делегат, и `this` в делегате — это сам делегат, не обёртка.
- `Delegates.observable` срабатывает **после** set, `vetoable` — **до** (можно отменить).

## Связанные темы

- [[q-03-scope-functions]]
- [[q-07-inline-crossinline]]

## Follow-up

- Что произойдёт, если в `by lazy` блоке кинется исключение?
- Как в class delegate переопределить часть методов и сохранить остальное?
