---
type: category-index
category: build-tooling
tags: [index, build-tooling]
---

# 10. Build & Tooling

Gradle, модуляризация, build variants, CI/CD, статический анализ.

## Подтемы

- Gradle: lifecycle, tasks, configurations, dependency resolution
- Gradle Kotlin DSL, version catalogs (`libs.versions.toml`)
- Build variants: flavors, build types, sourceSets
- Multi-module проекты: build performance, конвенции
- Configuration cache, build cache, parallel execution
- KSP vs KAPT, annotation processing
- R8/ProGuard: shrinking, optimization, obfuscation, rules
- APK vs AAB (Android App Bundle), dynamic delivery
- Signing config, keystores, Play App Signing
- CI: GitHub Actions / GitLab / Bitrise / Jenkins
- Fastlane (deploy, screenshots, metadata)
- Lint, Detekt, ktlint, Sonarqube
- Dependency management, BOMs, conflict resolution
- Composite builds, included builds
- Compose Compiler metrics

## Вопросы

```dataview
TABLE WITHOUT ID
  file.link AS "Вопрос",
  difficulty AS "Сложность",
  status AS "Статус",
  times_failed AS "Провалов"
FROM "01-Questions/10-Build-Tooling"
WHERE type = "question"
SORT difficulty DESC, status ASC
```
