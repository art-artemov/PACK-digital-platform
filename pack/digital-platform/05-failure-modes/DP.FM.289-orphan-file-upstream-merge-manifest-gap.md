---
id: DP.FM.289
name: "Orphan-файл из upstream merge без обновления manifest блокирует все коммиты в репо"
type: failure-mode
domain: DP
status: active
valid_from: 2026-07-16
source: "session-close 2026-07-14; bug cfa52cd, manifest-coverage hook"
related:
  see_also:
    - "DP.FM.287: update-sh merge erases user sections (смежно: upstream merge как источник неожиданных изменений)"
    - "DP.M.157: manifest-coverage-ci-check (метод, обнаруживающий этот класс gap — этот FM описывает последствие отсутствия escape hatch у проверки)"
tags: [iwe, manifest, pre-commit-hook, upstream-merge, orphan-file, git, gate, repo-health]
---

# DP.FM.289 — Orphan-файл из upstream merge блокирует все коммиты через manifest-coverage хук

## Паттерн

`git merge upstream/main` приносит новый файл. `update-manifest.json` не обновляется автоматически.
Manifest-coverage pre-commit хук обнаруживает файл без manifest-записи → **блокирует ЛЮБОЙ коммит в репо**, не только связанные с этим файлом.

## Сломанный flow

```
git merge upstream/main
  → scripts/gate-metrics.sh (новый файл)
  → update-manifest.json не обновлён
  → pre-commit: manifest-coverage check
  → ERROR: orphan file detected
  → ВСЕ коммиты заблокированы, включая исправление manifest
```

Репо заблокирован до ручного исправления manifest.

## Симптом

После upstream merge/sync любой `git commit` завершается с ошибкой manifest-coverage. Даже попытка зафиксировать исправление manifest — заблокирована (коммит уже требует manifest, которого нет).

## Диагностика

```bash
# Найти orphan-файлы (есть в репо, нет в manifest)
git ls-files | while read f; do
  grep -q "\"$f\"" update-manifest.json || echo "ORPHAN: $f"
done
```

## Fix

1. Добавить orphan-файл в `update-manifest.json` вручную
2. Зафиксировать: `git add update-manifest.json scripts/gate-metrics.sh && git commit`

## Правило

**После любого upstream merge — сразу проверить manifest на orphan-файлы до следующего коммита.**

Безопасный обход хука исключён (S-33 Hooks Bypass Gate). Исправление только через manifest.

## Системное решение

Manifest-gate должен либо:
- Auto-включать новые файлы через wildcard (relaxed mode),
- Либо иметь механизм «временного исключения для исправления самого manifest» (escape hatch).

## Применимость

Любая система с file manifest + strict pre-commit gate (IWE update-manifest.json, любые registry-based gatekeepers).
Срабатывает при upstream merge, cherry-pick, или `git checkout upstream/main -- path/to/file`.