---
type: question
category: architecture
difficulty: middle
tags: [solid, principles]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# SOLID на примерах Android

## Краткий ответ (TL;DR)

**S**ingle Responsibility — один класс, одна причина для изменения. **O**pen/Closed — открыт для расширения, закрыт для модификации. **L**iskov — наследник заменяет родителя без сюрпризов. **I**nterface Segregation — клиент не зависит от методов, которые не использует. **D**ependency Inversion — зависим от абстракций, не от деталей. На Android это: разделение Activity/VM/Repo, sealed interface для расширения, DI вместо singleton, мелкие repository interface вместо толстых.

## Развёрнутый ответ

### S — Single Responsibility

**Плохо**: Activity загружает данные, парсит JSON, рисует UI, навигирует.

```kotlin
// ❌
class ProfileActivity : AppCompatActivity() {
    override fun onCreate(...) {
        val response = OkHttpClient().newCall(Request.Builder().url(URL).build()).execute()
        val user = Gson().fromJson(response.body!!.string(), User::class.java)
        findViewById<TextView>(R.id.name).text = user.name
    }
}
```

**Хорошо**: разделено по обязанностям.

```kotlin
// ✅
class UserApi // network
class UserRepository(api: UserApi) // данные
class ProfileVm(repo: UserRepository) : ViewModel() // state
class ProfileActivity : ComponentActivity() // только UI
```

### O — Open/Closed

**Плохо**: класс знает все варианты, при добавлении нового — модификация.

```kotlin
// ❌
class Analytics {
    fun log(event: String, where: String) {
        if (where == "firebase") FirebaseAnalytics.log(event)
        else if (where == "amplitude") AmplitudeAnalytics.log(event)
    }
}
```

**Хорошо**: интерфейс + plug-in.

```kotlin
// ✅
interface AnalyticsTracker { fun log(event: String) }
class FirebaseTracker : AnalyticsTracker
class AmplitudeTracker : AnalyticsTracker

class Analytics(private val trackers: Set<AnalyticsTracker>) {
    fun log(event: String) = trackers.forEach { it.log(event) }
}
```

Новый трекер — новый класс, `Analytics` не трогаем. Используем DI multibinding.

### L — Liskov Substitution

**Плохо**: subclass нарушает контракт.

```kotlin
// ❌
open class Bird { open fun fly() }
class Penguin : Bird() {
    override fun fly() = throw UnsupportedOperationException()  // нарушение
}
```

**Хорошо**: правильная иерархия.

```kotlin
// ✅
interface Bird
interface FlyingBird : Bird { fun fly() }
class Penguin : Bird
class Eagle : FlyingBird { override fun fly() = ... }
```

Sealed классы помогают enforce'ить L через exhaustive `when`.

### I — Interface Segregation

**Плохо**: толстый интерфейс.

```kotlin
// ❌
interface UserRepository {
    suspend fun create(user: User)
    suspend fun read(id: Long): User
    suspend fun update(user: User)
    suspend fun delete(id: Long)
    suspend fun adminPurge()
    suspend fun reportAbuse(id: Long)
}

class ProfileVm(repo: UserRepository)  // нужен только read, но получает всё
```

**Хорошо**: разделить.

```kotlin
// ✅
interface UserReader { suspend fun read(id: Long): User }
interface UserWriter { suspend fun update(user: User) }
interface UserAdmin { suspend fun adminPurge() }

class UserRepositoryImpl : UserReader, UserWriter, UserAdmin

class ProfileVm(reader: UserReader)
```

Меньше mock в тестах, чёткие зоны ответственности.

### D — Dependency Inversion

**Плохо**: зависимость на конкретный класс.

```kotlin
// ❌
class UserVm : ViewModel() {
    private val repo = UserRepositoryImpl(RetrofitInstance.api, AppDb.dao)
    // высокоуровневая VM знает о Retrofit и Room
}
```

**Хорошо**: зависимость на интерфейс (через DI).

```kotlin
// ✅
class UserVm @Inject constructor(
    private val repo: UserRepository  // интерфейс, не impl
) : ViewModel()

// тесты подсунут FakeRepository
```

Это — основа Clean Architecture.

### Conceptual examples в Android

- **S**: Activity не должна делать сетевые вызовы.
- **O**: добавление новой analytics через DI multibinding, не изменением центрального класса.
- **L**: `Fragment` subclass не должен переопределять `onCreate` без `super.onCreate(savedInstanceState)`.
- **I**: разделить `UserRepository` на `UserReader`/`UserWriter`.
- **D**: ViewModel depends на `UserRepository` interface, реальный impl в `:data`.

### Code smells по SOLID

- **God Activity** (1000 строк) → нарушение S.
- **`is` checks с цепочкой** для разных типов → O нарушено, нужен polymorphism / sealed.
- **`throw UnsupportedOperationException()`** в overriding методе → L нарушено.
- **Один интерфейс с 20 методами** → I нарушено.
- **`new SomeImpl()` в высокоуровневом классе** → D нарушено, нужен DI.

## Подводные камни

- Чрезмерное SRP — каждый класс в одну строку → over-engineering, тяжело читать.
- ISP до абсурда: 10 интерфейсов по одному методу для одного repository → boilerplate без пользы.
- L нарушается тонко: `MutableList` ⊂ `List` — но `MutableList` для read даёт ту же гарантию? Не совсем (concurrent modification).
- DIP без DI-framework требует ручной фабрики → может усложнить.
- "SOLID" — guidelines, не догмы. Сильнее не значит лучше.

## Связанные темы

- [[q-03-clean-architecture]]
- [[q-06-hilt-dagger]]
- [[q-09-repository-pattern]]

## Follow-up

- Приведите пример нарушения Single Responsibility в Activity.
- Как Open/Closed реализуется через DI multibinding?
- Что такое Dependency Inversion и зачем она нужна?
