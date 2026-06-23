---
type: category-index
category: ui
tags: [index, ui]
---

# 3. UI (Compose + View)

Jetpack Compose, View system, custom views, анимации, навигация, темы.

## Подтемы

### Jetpack Compose
- Composable functions, recomposition, stability
- State: `remember`, `mutableStateOf`, `derivedStateOf`, `rememberSaveable`
- `CompositionLocal`, side effects (`LaunchedEffect`, `DisposableEffect`, `SideEffect`)
- Layouts, `Modifier`, custom layout, `SubcomposeLayout`
- Performance: skippable/restartable, `@Stable`, `@Immutable`
- Navigation Compose
- Compose + ViewSystem interop

### View system
- View lifecycle, draw cycle (measure/layout/draw)
- ViewGroup, custom View / ViewGroup
- RecyclerView: ViewHolder, DiffUtil, adapters, payload
- ConstraintLayout, MotionLayout
- Fragments, FragmentTransaction, child fragments
- Data Binding / View Binding
- Animations: property, transition, MotionLayout
- Themes, styles, attrs, day/night

## Вопросы

```dataview
TABLE WITHOUT ID
  file.link AS "Вопрос",
  difficulty AS "Сложность",
  status AS "Статус",
  times_failed AS "Провалов"
FROM "01-Questions/03-UI"
WHERE type = "question"
SORT difficulty DESC, status ASC
```
