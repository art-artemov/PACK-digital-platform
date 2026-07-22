---
id: DP.FM.284
name: "Хук-валидатор читает файл с диска вместо staged-версии"
type: failure-mode
domain: DP
status: active
valid_from: 2026-07-16
source: "session-close 2026-07-12; FMT-exocortex-template fix #248 (cec81a3)"
related:
  see_also:
    - "DP.FM.085: hook-installer-anti-patterns (смежно: хуки, другая ось)"
    - "DP.FM.038: validator-silent-pass (смежно: валидация, другой механизм)"
tags: [git, hooks, pre-commit, staged, validation, working-tree]
---

# DP.FM.284 — Хук-валидатор читает файл с диска вместо staged-версии

## Паттерн

Pre-commit хук читает артефакт напрямую (`cat path/to/file.md`), а не из staging area (`git show :path/to/file.md`). Хук проверяет рабочую копию на диске — которая может уже содержать следующее незастейженное изменение или не содержать staged-изменение (если файл отредактировали после `git add`).

**Симптом:** хук проверяет не то, что будет закоммичено.

## Диагностика

**Тест:** «После `git add file.md` отредактировал файл на диске (не делая `git add` снова) — хук видит новую версию?» Да → читает с диска, не из stage.

## Инцидент

FMT-exocortex-template, fix #248 (cec81a3): WeekPlan validator и DayPlan resolution оба читали с диска — баг проявился в двух независимых местах, что указывает на системный паттерн.

## Fix

```bash
# Неверно — читает рабочую копию:
content=$(cat path/to/file.md)

# Верно — читает staged-версию:
content=$(git show :path/to/file.md)

# Альтернатива — diff staged:
git diff --cached path/to/file.md
```

## Правило

**Pre-commit хук, проверяющий артефакт, ОБЯЗАН читать его из stage, а не с диска.**

При написании хука явно выбирать источник: staged (`git show :path`) vs working-tree (`cat path`) vs committed HEAD (`git show HEAD:path`).

## Применимость

Любые pre-commit хуки, валидирующие содержимое файлов: WeekPlan/DayPlan чеклисты, manifest-coverage, syntax validators.