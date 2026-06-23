---
type: question
category: android-core
difficulty: middle
tags: [lifecycle, activity]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Жизненный цикл Activity. Что между `onPause` и `onStop`? `onSaveInstanceState`?

## Краткий ответ (TL;DR)

`onCreate → onStart → onResume → (Running) → onPause → onStop → onDestroy`. `onPause` — Activity теряет фокус, но видна (overlay/dialog/multi-window split). `onStop` — полностью невидима. `onSaveInstanceState` вызывается перед `onStop` (с Android P) для сохранения UI-state в `Bundle` при возможной смерти процесса.

## Развёрнутый ответ

### Полный цикл

```
onCreate()       — создание, восстановление state из Bundle
  ↓
onStart()        — Activity видна, но ещё не интерактивна
  ↓
onResume()       — фокус, интерактивна
  ↓
[ RUNNING ]
  ↓
onPause()        — частично скрыта (dialog, multi-window, новая Activity сверху, но прозрачная)
  ↓
onStop()         — полностью скрыта
  ↓
onDestroy()      — finish() или kill процесса
```

При возврате: `onRestart → onStart → onResume`.

### Когда что вызывается

| Сценарий | Колбэки |
|----------|---------|
| Запуск новой Activity сверху (непрозрачной) | A: `onPause → onStop`; B: `onCreate → onStart → onResume` |
| Запуск прозрачной Activity / Dialog-themed | A: только `onPause` (НЕ `onStop`) |
| Кнопка Home | `onPause → onStop` (НЕ `onDestroy`) |
| Back с finish | `onPause → onStop → onDestroy` |
| Поворот экрана | `onPause → onStop → onDestroy → onCreate → onStart → onResume` |
| Configuration change с `android:configChanges` | НЕ пересоздаётся, вызывается `onConfigurationChanged` |
| Split-screen фокус не на нас | `onPause` (видна, не интерактивна) |
| Убит процесс в фоне | НЕТ колбэков |

### onSaveInstanceState

- Вызывается **только если система может убить процесс** — то есть в фоне или при config change.
- НЕ вызывается при явном `finish()` (пользователь сам закрыл — state не нужен).
- На Android P+ вызывается **после** `onStop` (раньше — до). Стало можно делать transactions во `onStop`.
- Парный `onRestoreInstanceState` после `onStart` (или Bundle приходит в `onCreate`).

```kotlin
override fun onSaveInstanceState(out: Bundle) {
    super.onSaveInstanceState(out)
    out.putString("query", searchView.query.toString())
}

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    savedInstanceState?.getString("query")?.let { searchView.setQuery(it, false) }
}
```

Лимит размера Bundle — 1MB на транзакцию (Binder). Большие данные → `ViewModel` + SavedStateHandle, либо persistence.

### Configuration changes

По умолчанию при повороте/смене темы/языка Activity пересоздаётся. Опции:
1. **Сохранять state** через `ViewModel` (живёт через configuration change) + `onSaveInstanceState` (для смерти процесса).
2. **Перехватить config** в манифесте: `android:configChanges="orientation|screenSize"` → вызывается `onConfigurationChanged`, Activity не пересоздаётся. **Не рекомендуется** — много граблей с ресурсами.

### Important caveats

- `onPause` должен быть **быстрым** (< 500ms) — иначе ANR при переходе.
- В `onStop` нельзя делать `FragmentTransaction.commit()` до Android P → `commitAllowingStateLoss` или ждать `onStart`.
- `isFinishing` отличает finish от config change.
- `isChangingConfigurations` подтверждает config change.

## Подводные камни

- Деление «activity видима» vs «activity в фоне» не очевидно: dialog-themed activity сверху → ты в `onPause`, но не в `onStop`.
- Listener'ы регистрировать в `onStart`, отписывать в `onStop` (а не on Resume/Pause) — для multi-window правильнее.
- Не сохранять Parcelable размером > 50KB в Bundle.
- `ViewModel.onCleared()` не гарантирован при kill процесса.

## Связанные темы

- [[q-02-fragment-lifecycle]]
- [[q-03-context]]
- [[q-09-process-death]]

## Follow-up

- Чем `onPause` отличается от `onStop` в multi-window режиме?
- Что произойдёт с ViewModel при повороте экрана? А при kill процесса?
- Когда не вызывается `onSaveInstanceState`?
