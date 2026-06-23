---
type: question
category: android-core
difficulty: senior
tags: [process-death, state, savedstate]
created: 2026-06-16
last_reviewed: 2026-06-16
times_failed: 1
times_passed: 0
status: learning
---

# Process death: что переживает, что нет. ViewModel vs SavedStateHandle vs onSaveInstanceState

## Краткий ответ (TL;DR)

Когда система убивает процесс в фоне, **ViewModel умирает** (он живёт только в памяти). Переживает только то, что записано в **Bundle** через `onSaveInstanceState`/`SavedStateHandle` или в **persistent storage** (DataStore, Room, SharedPreferences, файлы). При возврате пользователя в приложение Activity воссоздаётся с этим Bundle, но процесс — новый.

## Развёрнутый ответ

### Что такое process death

В отличие от configuration change (поворот), это полный kill процесса системой (out-of-memory, разлогирована модель LRU). Признаки:
- В Logcat не видно ничего особенного, новый процесс просто стартует.
- Application.onCreate() вызывается заново.
- Все singleton'ы / ViewModel созданы с нуля.
- Activity воссоздаётся **только если** пользователь возвращается в неё (через Recents).

### Что переживает

| Тип хранения | Config change | Process death |
|--------------|---------------|---------------|
| `ViewModel` (in-memory) | ✅ | ❌ |
| `onSaveInstanceState` Bundle | ✅ | ✅ |
| `SavedStateHandle` в ViewModel | ✅ | ✅ |
| `Activity.lateinit var` | ❌ | ❌ |
| `Application.field` | ✅ | ❌ |
| Static / `object` | ✅ | ❌ |
| `SharedPreferences` / DataStore / Room / File | ✅ | ✅ |

### SavedStateHandle

```kotlin
class MyViewModel(private val state: SavedStateHandle) : ViewModel() {
    private val query: StateFlow<String> = state.getStateFlow("query", "")

    fun setQuery(q: String) { state["query"] = q }
}
```

- `SavedStateHandle` — типа `Bundle`, инжектится в ViewModel (Hilt: `@HiltViewModel` делает автоматически).
- Используется и для **navigation arguments** (Compose Navigation, Safe Args).
- Лимит размера — те же ~1MB Binder transaction.

### Когда что использовать

| Нужно сохранить | Где |
|-----------------|-----|
| Текущий scroll, выбранный таб, query в поиске | SavedStateHandle / Bundle |
| Список с сервера (можно перезагрузить) | ViewModel (in-memory cache) |
| Драфт текста, который пользователь набрал | SavedStateHandle + autosave в DataStore |
| Токен авторизации, настройки | DataStore / EncryptedSharedPreferences |
| Кэш картинок | Disk cache (Coil/Glide) |

### Как воспроизвести process death

1. Settings → Developer Options → **Don't keep activities** — Activity убивается при `onStop`.
2. ADB: `adb shell am kill <package>` (только для backgrounded app).
3. Из Logcat: `adb shell am force-stop <package>`.

### Compose + savedstate

```kotlin
val query = rememberSaveable { mutableStateOf("") }
```

`rememberSaveable` сохраняет через `Saver` (по умолчанию для Parcelable/primitive). Для кастомных типов — свой `Saver`. Под капотом — тот же `SavedStateRegistry`.

### Don't keep activities

Жёстокий тест, но не симулирует ВСЕ аспекты process death:
- ✅ Активити умирает.
- ❌ Application НЕ пересоздаётся.
- ❌ ViewModel НЕ пересоздаётся (он привязан к ViewModelStoreOwner — NavBackStackEntry/Activity, а Activity воссоздаётся в том же процессе).

Реальный process death симулируется только kill процесса.

## Подводные камни

- Singleton кэш с in-memory данными → после process death пусто → нужно учитывать в дизайне.
- `SavedStateHandle` для больших Parcelable — `TransactionTooLargeException`.
- Compose `remember { }` без `rememberSaveable` теряется при config change тоже.
- `Application.onCreate()` после process death — точка холодного старта; init heavy SDK тут → плохой first frame time.
- `viewModelScope` отменяется при `onCleared`, но `onCleared` НЕ вызывается при kill процесса.

## Связанные темы

- [[q-01-activity-lifecycle]]
- [[../04-Architecture/q-viewmodel]]
- [[../09-Compose/q-state-saving]]

## Follow-up

- Чем `SavedStateHandle` отличается от `onSaveInstanceState`?
- Что произойдёт с подпиской `viewModelScope.launch` при process death?
- Как протестировать process death в реальном устройстве?
