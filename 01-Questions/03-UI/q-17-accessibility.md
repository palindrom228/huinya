---
type: question
category: ui
difficulty: middle
tags: [accessibility, talkback, a11y]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Accessibility: TalkBack, contentDescription, semantics в Compose

## Краткий ответ (TL;DR)

A11y — обязательная часть production-приложения. View: `contentDescription`, `importantForAccessibility`, `AccessibilityDelegate`. Compose: `Modifier.semantics { ... }` + `contentDescription`. Минимум для кнопок-иконок — описание; для декоративных — `null`/`Role.None`. Проверять через TalkBack, Accessibility Scanner, тач target ≥ 48dp.

## Развёрнутый ответ

### View

```kotlin
imageButton.contentDescription = "Send message"
decorativeImage.importantForAccessibility = View.IMPORTANT_FOR_ACCESSIBILITY_NO
```

`AccessibilityDelegate` (см. [[q-16-custom-view]]) для custom view с кастомной семантикой.

### Compose

```kotlin
Image(
    painter = painterResource(R.drawable.ic_send),
    contentDescription = "Send"     // обязательно для интерактивных
)

Image(
    painter = painterResource(R.drawable.bg_pattern),
    contentDescription = null       // декоративное → TalkBack пропустит
)

Box(
    Modifier
        .clickable(onClickLabel = "Open profile") { open() }
        .semantics { role = Role.Button }
)
```

### Semantics в Compose

```kotlin
Modifier.semantics {
    contentDescription = "Volume slider, $value percent"
    role = Role.Switch
    stateDescription = if (checked) "on" else "off"
    progressBarRangeInfo = ProgressBarRangeInfo(value, 0f..100f)
    onClick(label = "Toggle") { toggle(); true }
}
```

`mergeDescendants = true` — собирает семантику детей в один node (для строк-карточек).

```kotlin
Row(Modifier.clickable { open() }.semantics(mergeDescendants = true) {}) {
    Image(..., contentDescription = null)
    Text("Jane Doe")
    Text("Last seen 5 min ago")
}
// TalkBack прочтёт "Jane Doe Last seen 5 min ago, double tap to activate"
```

### Touch target size

Минимум **48×48 dp** для интерактивных элементов. В Compose Material — auto-padded, но в custom — следить.

```kotlin
Box(
    Modifier
        .size(24.dp)        // визуальный
        .clickable { ... }
        .sizeIn(minWidth = 48.dp, minHeight = 48.dp)  // touch target
)
```

### Контраст

WCAG AA: ratio 4.5:1 для обычного текста, 3:1 для крупного. Android Studio Accessibility Scanner проверяет.

### Live regions

Для динамически меняющегося контента (timer, error):

```kotlin
Text(
    text = "Error: $error",
    modifier = Modifier.semantics { liveRegion = LiveRegionMode.Polite }
)
```

`Polite` — TalkBack дочитает текущее и потом сообщит; `Assertive` — прервёт.

### testTag и semantics

```kotlin
Modifier.testTag("loginButton").semantics { contentDescription = "Log in" }
```

`testTag` — для UI-тестов; не озвучивается TalkBack по умолчанию. Через `useUnmergedTree` в тестах можно найти.

### Headings

```kotlin
Text("Section",
    modifier = Modifier.semantics { heading() },
    style = MaterialTheme.typography.titleLarge
)
```

TalkBack может навигировать по headings.

### Custom actions

```kotlin
Modifier.semantics {
    customActions = listOf(
        CustomAccessibilityAction("Delete") { delete(); true },
        CustomAccessibilityAction("Archive") { archive(); true }
    )
}
```

Альтернатива swipe-actions, доступная для TalkBack.

### Тестирование

- **TalkBack**: Settings → Accessibility → TalkBack.
- **Accessibility Scanner**: app from Play Store, проверяет touch size, contrast, описания.
- **Layout Inspector** → Show Accessibility Properties.
- **espresso-accessibility**: `AccessibilityChecks.enable()` в Espresso тестах.

## Подводные камни

- `Icon(...)` в Compose требует `contentDescription` — `null` только для чисто декоративных.
- `mergeDescendants` без `clickable` → дочерние node всё равно отдельные.
- `Modifier.size(24.dp).clickable { }` без min 48 → Accessibility Scanner ругается.
- Text над тёмным фоном без контраста → fail.
- В Compose Material3 `Button` автоматически имеет роль `Button` — не дублировать `role = Button` в `semantics`.

## Связанные темы

- [[q-16-custom-view]]
- [[q-01-compose-mental-model]]
- [[../12-Quality/q-ui-tests]]

## Follow-up

- Когда `contentDescription = null` корректно?
- Зачем `mergeDescendants` в Compose semantics?
- Какой минимальный touch target по a11y?
