---
id: DP.M.396
name: "Разделение iOS entitlements: отдельный файл для Debug и Release"
type: method
domain: digital-platform / mobile-ios
status: active
valid_from: 2026-07-16
sources:
  - "WP-13 mobile-new deeplinks, session-close 2026-07-14 (Xcode project fix)"
related:
  addresses: [DP.FM.285]
tags: [ios, xcode, entitlements, debug-release, universal-links, project-structure]
---

# DP.M.396 — Разделение iOS entitlements: отдельный файл для Debug и Release

## Определение

Создать два отдельных `.entitlements`-файла для iOS-проекта и привязать каждый к своей build configuration в `project.pbxproj`.

## IPO

- **Вход:** Xcode-проект с единым `.entitlements`-файлом (стандарт после создания из шаблона)
- **Процесс:** добавить второй entitlements-файл, привязать в pbxproj, проверить `plutil -lint`
- **Выход:** Debug и Release могут иметь разные записи (Push Environment, Associated Domains mode) без взаимного влияния

## Шаги

1. **Создать Debug-entitlements:** `humdesApp/humdesApp.entitlements` (или `Debug.entitlements`)
2. **Перенести** Debug-специфичные записи: `?mode=developer` у `applinks:`, Push Environment = `development`
3. **В `project.pbxproj`** для Debug-конфигурации заменить `CODE_SIGN_ENTITLEMENTS` на новый файл
4. **Release** остаётся на исходном файле (без `?mode=developer`, Push Environment = `production`)
5. **Зарегистрировать** новый файл в `PBXFileReference` и группе проекта
6. **Проверить:** `plutil -lint project.pbxproj` — должен завершиться без ошибок

## Структура результата

```
humdesApp/
├── humdesApp.entitlements          # Debug: ?mode=developer, Push=development
└── humdesAppRelease.entitlements   # Release: чистый, Push=production
```

В `project.pbxproj`:
```
/* Debug */
CODE_SIGN_ENTITLEMENTS = humdesApp/humdesApp.entitlements;
/* Release */
CODE_SIGN_ENTITLEMENTS = humdesApp/humdesAppRelease.entitlements;
```

## Правило

**При создании нового Xcode-проекта — сразу разделять Debug и Release entitlements.**

Один entitlements-файл для обоих = неявная зависимость: Dev Mode флаги попадают в боевую сборку.

## Связи

- [DP.FM.285](../05-failure-modes/DP.FM.285-ios-mode-developer-entitlements-universal-links.md) — failure mode, который этот метод устраняет (`?mode=developer` в Release)