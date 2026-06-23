---
type: question
category: android-core
difficulty: middle
tags: [deeplinks, app-links, navigation]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Deep links vs App Links vs Custom schemes. Verification, navigation, безопасность

## Краткий ответ (TL;DR)

**Deep link** — `Intent` с URI, который ваше приложение умеет обработать (`https://example.com/x` или `myapp://x`). **App Links** (Android 6+) — HTTPS deep links, **верифицированные** через `assetlinks.json` на сервере — открываются в вашем приложении без chooser'а. **Custom schemes** (`myapp://`) — небезопасно (любой может claim same scheme).

## Развёрнутый ответ

### Объявление intent-filter

```xml
<activity android:name=".DeepLinkActivity" android:exported="true">
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https"
              android:host="example.com"
              android:pathPrefix="/product/" />
    </intent-filter>
</activity>
```

`android:autoVerify="true"` + правильный `assetlinks.json` = App Links.

### assetlinks.json

```
https://example.com/.well-known/assetlinks.json
```

```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example.app",
    "sha256_cert_fingerprints": ["XX:XX:..."]
  }
}]
```

Должен быть доступен по HTTPS без редиректов, с правильным content-type. Проверить:

```bash
adb shell pm verify-app-links --re-verify com.example.app
adb shell pm get-app-links com.example.app
```

### Custom scheme

```xml
<data android:scheme="myapp" android:host="open" />
```

`myapp://open?id=42`. **Проблема**: любое приложение может объявить такой же scheme → перехват. Для secure flows (OAuth callback) — не использовать.

### Извлечение данных

```kotlin
val data: Uri? = intent.data
val id = data?.getQueryParameter("id")
val pathSegments = data?.pathSegments
```

### Compose Navigation

```kotlin
NavHost(navController, startDestination = "home") {
    composable(
        "product/{id}",
        deepLinks = listOf(navDeepLink { uriPattern = "https://example.com/product/{id}" })
    ) { backstack ->
        val id = backstack.arguments?.getString("id")
        ProductScreen(id)
    }
}
```

### Безопасность

1. **Никогда не доверяй данным из deep link**. Validate id/UUID, escape для SQL/HTML.
2. **Auth flows**: code injection через `state` parameter manipulation.
3. **WebView + javascript:** scheme — XSS поверхность.
4. **Intent redirection**: deep link с extras-Intent, который потом запускается → атака.
5. **Implicit intents**: проверь `getCallingActivity()` если важен origin (но это легко обходится).

### OAuth + App Links

Современная схема: HTTPS App Links вместо custom schemes для callback. RFC 8252 (OAuth for Native Apps).

```
Authorization Endpoint redirect → https://example.com/oauth-callback
  → App Links → MainActivity → handle code
```

### Debug

```bash
adb shell am start -W -a android.intent.action.VIEW \
    -d "https://example.com/product/42" com.example.app
```

### Disambiguation dialog

Если несколько приложений claim same URI и App Links не верифицирован → пользователь видит chooser. Можно `Intent.createChooser` для явного выбора, но это не для App Links flow.

## Подводные камни

- `autoVerify` без `assetlinks.json` → ничего не сломает, но App Links не активируется.
- Кеш верификации Android-системы — иногда нужно очищать (`pm verify-app-links --re-verify`).
- `pathPattern` использует свой regex синтаксис (не PCRE) — gotcha.
- Custom scheme без `BROWSABLE` category → не открывается из браузера.
- Сертификат подписи изменился (debug → release) → App Links перестают верифицироваться.
- App Bundles с Play App Signing → нужен fingerprint **upload key**, не локальный.

## Связанные темы

- [[q-04-intents]]
- [[../11-Security/q-oauth-pkce]]
- [[../09-Compose/q-navigation]]

## Follow-up

- Чем App Links отличаются от deep links?
- Почему custom scheme — небезопасный выбор для OAuth callback?
- Что произойдёт, если изменить подпись приложения?
