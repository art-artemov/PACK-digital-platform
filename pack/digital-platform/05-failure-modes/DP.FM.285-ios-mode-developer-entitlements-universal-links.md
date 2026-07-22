---
id: DP.FM.285
name: "?mode=developer в entitlements боевой iOS-сборки блокирует Universal Links"
type: failure-mode
domain: DP
status: active
valid_from: 2026-07-16
source: "session-close 2026-07-14; WP-13 mobile-new deeplinks диагностика (Xcode entitlements)"
tags: [ios, xcode, entitlements, universal-links, associated-domains, mobile, release-config]
---

# DP.FM.285 — `?mode=developer` в entitlements боевой iOS-сборки блокирует Universal Links

## Паттерн

Xcode добавляет `?mode=developer` к записи в `Associated Domains` entitlements при включённом «Associated Domains Development Mode». На устройствах разработчика это обходит CDN-верификацию AASA. В боевой (TestFlight/AppStore) сборке — iOS не подтверждает связь приложения с доменом → тап по Universal Link открывается в Safari, а не в приложении.

## Диагностика

**Тест:** «`mobile` (старый проект) работает, `mobile-new` (новый проект) — нет, AASA и Bundle ID одинаковы?» → Проверить entitlements в `project.pbxproj` для Release-конфигурации.

```bash
# Симптом в entitlements:
applinks:humdes.com?mode=developer   # ← неверно для Release

# Должно быть:
applinks:humdes.com                  # ← верно
```

## Инцидент

WP-13 (mobile-new deeplinks): Universal Links регрессировали между `mobile` и `mobile-new` несмотря на одинаковые AASA и Bundle ID. Причина: Xcode создаёт новый проект с единым `.entitlements`-файлом для Debug и Release — флаг `?mode=developer` попал в Release.

## Fix

Отдельный entitlements-файл для Debug-конфигурации (см. DP.M.396). Release-конфигурация в `project.pbxproj` ссылается только на Release-entitlements без флага.

## Правило

**В Release-entitlements записи `applinks:` НЕ должны содержать `?mode=developer`.**

При создании нового Xcode-проекта — явно разделить Debug и Release entitlements.

## Применимость

Любое iOS-приложение с Universal Links, созданное из шаблона Xcode (один entitlements-файл по умолчанию).