---
id: DP.FM.288
name: "CLAUDE_SESSION_ID не экспортирован в хуки — sentinel-механизм не защищает"
type: failure-mode
domain: DP
status: active
valid_from: 2026-07-16
source: "session-close 2026-07-14; /audit-installation, bug f9cb10a"
related:
  see_also:
    - "DP.FM.009: hardcoded script path (смежно: IWE tooling)"
tags: [iwe, claude-code, hooks, pre-tool-use, claude-session-id, sentinel, dry-run, security]
---

# DP.FM.288 — CLAUDE_SESSION_ID не экспортируется в хуки: sentinel-механизм не защищает

## Паттерн

`CLAUDE_SESSION_ID` **не пробрасывается** ни в Bash-инструмент агента, ни в окружение PreToolUse-хуков в Claude Code CLI (macOS).

Sentinel-механизм, который должен связывать «скилл создал флаг» ↔ «хук проверил флаг», **никогда не совпадает по имени файла**.

## Сломанный flow

```
Скилл:   CLAUDE_SESSION_ID="" → uuidgen() → /tmp/iwe-dry-run-UUID.flag
Хук:     ${CLAUDE_SESSION_ID:-noid} → /tmp/iwe-dry-run-noid.flag
Результат: файлы разные → проверка "allow" срабатывает всегда → sentinel не защищает
```

## Симптом

Sentinel-механизм dry-run (WP-265 Ф5.2) молча «работает» — хук не блокирует никаких операций, которые должен блокировать. `/audit-installation` (VR.R.002) выносит вердикт ❌ по разделу «Ритуалы» именно из-за этого.

## Диагностика

```bash
echo $CLAUDE_SESSION_ID   # В Bash-инструменте агента: пусто
env | grep CLAUDE          # В хуке: переменная отсутствует
```

## Fix-варианты

1. **Минимальный:** скилл создаёт sentinel явно с именем `noid` (`/tmp/iwe-dry-run-noid.flag`) — совпадёт с fallback хука при всегда-пустой переменной
2. **Правильный:** выяснить, можно ли прокинуть `CLAUDE_SESSION_ID` через `settings.json` env-блок
3. **Документационный:** предупреждение в SKILL.md скилла (Шаг 2.5) о пустом fallback

## Правило

**Нельзя полагаться на `CLAUDE_SESSION_ID` для координации скилл ↔ хук в Claude Code CLI.**

Любой механизм, использующий эту переменную для синхронизации, работает только в среде, где переменная явно пробрасывается.

## Применимость

Claude Code CLI (macOS, v0.35.x). IWE-скиллы с dry-run-контрактом (audit-installation, любой скилл через sentinel-механику WP-265).