---
type: category-index
category: security
tags: [index, security]
---

# 11. Security

Хранение секретов, сеть, обфускация, root-детект, биометрия, OWASP Mobile.

## Подтемы

- Android Keystore: KeyGenParameterSpec, hardware-backed keys, StrongBox
- EncryptedSharedPreferences, EncryptedFile, Tink
- Network security config, certificate pinning, mTLS
- Безопасное использование WebView (JS interface, mixed content)
- Биометрия: BiometricPrompt, fallback policy
- ProGuard / R8 как защита (и почему это не серебряная пуля)
- Root detection, SafetyNet → Play Integrity API
- Tamper detection, code integrity
- Secrets в коде: BuildConfig, NDK, server-side
- OAuth flow в мобильном: PKCE, refresh-token
- Deep links и App Links — security implications
- Intent redirection, exported components
- OWASP Mobile Top 10
- GDPR, согласия, аналитика и приватность

## Вопросы

```dataview
TABLE WITHOUT ID
  file.link AS "Вопрос",
  difficulty AS "Сложность",
  status AS "Статус",
  times_failed AS "Провалов"
FROM "01-Questions/11-Security"
WHERE type = "question"
SORT difficulty DESC, status ASC
```
