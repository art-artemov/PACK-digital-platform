---
id: DP.FM.290
name: "Задекларированный fail-safe не подключён в конфигурации хуков"
type: failure-mode
domain: DP
status: active
valid_from: 2026-07-16
source: "session-close 2026-07-16; WP-395 Status Reporting; upstream issue #266"
related:
  see_also:
    - "DP.FM.288: CLAUDE_SESSION_ID не экспортирован в хуки (смежный класс: декларация ≠ реальность)"
tags: [iwe-tooling, hooks, configuration, multi-agent, fail-safe, wiring]
---

# DP.FM.290 — Задекларированный fail-safe не подключён в хуках

## Паттерн

Инструкция (CLAUDE.md, SKILL.md) обещает автоматический страховочный механизм («если агент не отчитался сам — детерминированно отчитается хук»). Механизм фактически не зарегистрирован в `hooks/` или `settings.json`. При отказе основного пути страховка молча не срабатывает.

**Ключевое:** отказ страховки не имеет собственного симптома → невидим до инцидента.

## Диагностика

```bash
# Проверить, реально ли зарегистрирован обещанный механизм
grep -r "<имя_скрипта_или_хука>" .claude/hooks/
grep -r "<имя_скрипта_или_хука>" .claude/settings.json
```

Аудит через `grep` по конфигурации хуков — не чтение документации.

## Инцидент

CLAUDE.md (§ Status Reporting) обещал fail-safe `agent-status-report.sh` из Stop-хука — скрипт существовал в шаблоне, но в `.claude/hooks/` не было ни одной ссылки на него. Fail-safe молча не работал неизвестно сколько времени.

## Правило

«Declared capability = wired capability» — не по умолчанию, а только после явной проверки конфигурации.

## Связи

- Смежно: [DP.FM.288](DP.FM.288-session-id-not-exported-to-hooks.md) (CLAUDE_SESSION_ID не экспортирован в окружение хуков)
- Дополняет: [DP.M.398](../03-methods/DP.M.398-git-history-check-before-fixing-paths.md) (проверка git-истории до починки путей — тот же класс: декларация ≠ реальность)