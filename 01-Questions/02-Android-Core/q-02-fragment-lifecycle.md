---
type: question
category: android-core
difficulty: senior
tags: [lifecycle, fragment, viewlifecycle]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Жизненный цикл Fragment и View Lifecycle. Почему ViewBinding надо чистить?

## Краткий ответ (TL;DR)

Fragment имеет **два** lifecycle: Fragment lifecycle (`onCreate ... onDestroy`) и View lifecycle (`onCreateView ... onDestroyView`). View может пересоздаваться без пересоздания фрагмента (backstack, `replace` с `addToBackStack`). Поэтому ссылки на View (включая binding) надо обнулять в `onDestroyView`, иначе утечка View → Activity → Context.

## Развёрнутый ответ

### Два жизненных цикла

```
Fragment:
  onAttach → onCreate → onCreateView → onViewCreated → onStart → onResume
   ↓
  onPause → onStop → onDestroyView → onDestroy → onDetach

View lifecycle (viewLifecycleOwner):
  starts at onCreateView (когда возвращён View ≠ null)
  ends at onDestroyView
```

### Когда View умирает без Fragment

```kotlin
fragmentManager.beginTransaction()
    .replace(R.id.container, fragmentA)
    .addToBackStack(null)
    .commit()
```

После `replace` фрагмент A: `onPause → onStop → onDestroyView` (но **НЕ** `onDestroy`). View уничтожается. Если пользователь нажмёт Back → восстановится: `onCreateView → onViewCreated → ...`.

То же при `Fragment.setMaxLifecycle(STARTED)` — View может пересоздаваться.

### Binding cleanup

```kotlin
class MyFragment : Fragment(R.layout.frag) {
    private var _binding: FragBinding? = null
    private val binding get() = _binding!!

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        _binding = FragBinding.bind(view)
        // use binding
    }

    override fun onDestroyView() {
        _binding = null   // ← обязательно
        super.onDestroyView()
    }
}
```

Альтернатива — delegate (`viewBindingProperty()` от kirich или собственный).

### viewLifecycleOwner vs Fragment

```kotlin
// ПРАВИЛЬНО
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(STARTED) {
        viewModel.state.collect { binding.title.text = it.title }
    }
}

// ОПАСНО
lifecycleScope.launch {
    viewModel.state.collect { binding.title.text = it.title }  // binding == null после onDestroyView!
}
```

`lifecycleScope` живёт до `onDestroy` фрагмента (мог пережить уничтожение View), а `viewLifecycleOwner.lifecycleScope` — до `onDestroyView`.

### Главные правила

1. Все подписки на View (collect, observe) — через `viewLifecycleOwner`.
2. `LiveData.observe(viewLifecycleOwner, ...)` — НЕ `observe(this, ...)`. Иначе при возврате из backstack создастся **вторая** подписка → дубли.
3. Binding обнулять в `onDestroyView`.
4. Нельзя обращаться к `requireView()` после `onDestroyView`.

### childFragmentManager vs parentFragmentManager

- `childFragmentManager` — фрагменты внутри этого фрагмента.
- `parentFragmentManager` — менеджер родителя (Activity или родительского фрагмента).
- Путать → транзакции уходят не туда, фрагменты пропадают при ротации.

### Fragment Result API

С AndroidX замена `setTargetFragment`:

```kotlin
parentFragmentManager.setFragmentResultListener("req", this) { key, bundle -> ... }
// в другом фрагменте:
parentFragmentManager.setFragmentResult("req", bundleOf("value" to 42))
```

Lifecycle-aware: listener активен только в STARTED.

## Подводные камни

- `_binding = null` забыл → утечка всей View hierarchy (через каждое поле binding).
- Подписка на LiveData через `this` (а не `viewLifecycleOwner`) → при возврате из backstack дубли observer'ов.
- `requireView()` после `onDestroyView` → `IllegalStateException`.
- `commit` транзакций в `onStop` до Android P → `IllegalStateException`. Используй `commitAllowingStateLoss` осознанно.

## Связанные темы

- [[q-01-activity-lifecycle]]
- [[q-09-process-death]]

## Follow-up

- Почему наблюдение LiveData через `this` во фрагменте создаёт дубли?
- Чем `viewLifecycleOwner.lifecycleScope` отличается от `lifecycleScope`?
- Что произойдёт с ViewModel фрагмента после `onDestroyView`?
