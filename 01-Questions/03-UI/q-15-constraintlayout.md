---
type: question
category: ui
difficulty: middle
tags: [constraint-layout, view-system]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# ConstraintLayout: chains, barriers, guidelines, MotionLayout

## Краткий ответ (TL;DR)

ConstraintLayout — flat ViewGroup, описывающий позиции через **constraints** к другим view / parent. Заменяет вложенные LinearLayout/RelativeLayout (хороший perf — один pass). Powerful tools: **chains** (равномерное распределение), **barriers** (виртуальные границы по группе view), **guidelines** (виртуальные линии в %). **MotionLayout** = ConstraintLayout + анимации между двумя ConstraintSet.

## Развёрнутый ответ

### Базовые constraints

```xml
<TextView
    android:id="@+id/title"
    app:layout_constraintTop_toTopOf="parent"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintEnd_toEndOf="parent" />
```

Каждой view нужны constraints **по обоим осям** (X и Y), иначе размещается в (0,0).

### Размеры

- `0dp` = `match_constraint` (растягивается между constraint'ами).
- `wrap_content` / конкретный dp.
- `app:layout_constraintWidth_percent="0.5"` — половина parent.
- `app:layout_constraintWidth_default="wrap"` + `min/max` — ограничения.

### Chains

Группа view связана взаимно по одной оси:

```xml
<!-- horizontal chain: A → B → C -->
<View android:id="@+id/a"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintEnd_toStartOf="@id/b" />
<View android:id="@+id/b"
    app:layout_constraintStart_toEndOf="@id/a"
    app:layout_constraintEnd_toStartOf="@id/c" />
<View android:id="@+id/c"
    app:layout_constraintStart_toEndOf="@id/b"
    app:layout_constraintEnd_toEndOf="parent" />
```

Стили (на первом элементе chain'а):
- `spread` (default) — равные отступы.
- `spread_inside` — крайние прижаты, средние распределены.
- `packed` — все вместе, отступы по краям.
- `weighted` (с `0dp` + `layout_constraintHorizontal_weight`) — как LinearLayout weight.

### Barriers

Виртуальная граница по краю **самой широкой/высокой** view из группы:

```xml
<androidx.constraintlayout.widget.Barrier
    android:id="@+id/barrier"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    app:barrierDirection="end"
    app:constraint_referenced_ids="label1,label2,label3" />

<TextView app:layout_constraintStart_toEndOf="@id/barrier" />
```

Полезно когда тексты разной длины — value column всегда начинается за самой длинной label.

### Guidelines

```xml
<androidx.constraintlayout.widget.Guideline
    android:id="@+id/g1"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    app:layout_constraintGuide_percent="0.5" />
```

Невидимая линия, к которой можно привязываться. Полезно для адаптивных layout (50/50, 30/70).

### Groups

```xml
<androidx.constraintlayout.widget.Group
    android:id="@+id/loading"
    android:visibility="gone"
    app:constraint_referenced_ids="progress,text1,text2" />
```

Управление видимостью группы view без вложения.

### Bias

```xml
app:layout_constraintHorizontal_bias="0.3"
```

При наличии двух противоположных constraint'ов — bias 0..1 определяет позицию (0.3 = ближе к началу).

### Aspect ratio

```xml
android:layout_width="0dp"
android:layout_height="0dp"
app:layout_constraintDimensionRatio="16:9"
```

### MotionLayout

```xml
<androidx.constraintlayout.motion.widget.MotionLayout
    app:layoutDescription="@xml/scene_main">
    <!-- views -->
</MotionLayout>
```

```xml
<!-- scene_main.xml -->
<MotionScene>
    <Transition app:constraintSetStart="@id/start"
                app:constraintSetEnd="@id/end"
                app:duration="300">
        <OnSwipe app:touchAnchorId="@id/handle"
                 app:dragDirection="dragUp" />
    </Transition>
    <ConstraintSet android:id="@+id/start"> ... </ConstraintSet>
    <ConstraintSet android:id="@+id/end"> ... </ConstraintSet>
</MotionScene>
```

Анимирует между двумя ConstraintSet — swipe, click, programmatic progress. Powerful, но XML-тяжёлый. В Compose обычно проще через `animate*AsState` + `Modifier.layoutId`.

### Performance

ConstraintLayout быстрее вложенных LinearLayout в **flat-древе**: один measure pass, нет двойного measure детей. Но **deeply nested ConstraintLayout** — не быстрее. Лучше один большой flat layout.

## Подводные камни

- View без constraint по одной оси → размещается в (0,0), `tools:layout_editor_absoluteX/Y` в редакторе вводит в заблуждение.
- `wrap_content` + `app:layout_constrainedWidth="true"` нужно, чтобы wrap уважал constraints (иначе вылезает за границы).
- Барьер `referenced_ids` не работает с динамически добавляемыми view (`barrier.update()`).
- MotionLayout — отладка сложная, лучше Layout Inspector + LayoutDescription editor.
- `0dp` ≠ `match_parent` в CL: это `match_constraint`.

## Связанные темы

- [[q-13-views-measure-layout-draw]]
- [[q-10-compose-animations]]
- [[q-19-motion-layout]]

## Follow-up

- Чем `Barrier` отличается от `Guideline`?
- Зачем `0dp` в ConstraintLayout?
- Когда MotionLayout лучше property animations?
