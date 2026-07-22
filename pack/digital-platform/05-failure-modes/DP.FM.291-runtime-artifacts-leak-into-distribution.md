---
id: DP.FM.291
name: "Runtime-артефакты утекают в дистрибутив через git-отслеживаемую директорию без gitignore"
type: failure-mode
domain: DP
status: active
valid_from: 2026-07-16
source: "session-close 2026-07-16; update v0.35.5; upstream issue #266 п.3"
related:
  see_also:
    - "DP.FM.289: orphan-файл из upstream merge без обновления manifest (смежно: поставка добавляет лишние файлы)"
    - "DP.FM.287: 3-way merge шаблона не различает L1/L2/L3 (смежно: update.sh вносит нежелательное)"
tags: [template, distribution, gitignore, runtime-artifacts, iwe-tooling, delivery]
---

# DP.FM.291 — Runtime-артефакты утекают в дистрибутив

## Паттерн

Инструмент пишет runtime-выхлоп (лог, state-файл) в директорию, которая git-отслеживается в авторском шаблоне → файл закоммичен и попадает в релиз → каждый пользователь получает чужой runtime-артефакт при обновлении через `update.sh`.

## Симптом

После `update.sh` в рабочей директории появляются файлы с timestamp-именами (дата, UUID, хеш) — признак runtime-артефакта из авторского окружения.

## Инцидент

Скилл `/audit-installation` писал лог (`iwe-audit-20260715-175936.log`) в директорию `IWE/scripts/` внутри git-отслеживаемого шаблона → лог закоммичен в релиз v0.35.5 → тиражируется всем пользователям при update.

## Двухслойный fix-паттерн

**Уровень инструмента:**
- runtime-выхлоп пишется в директорию с `.gitignore: *`
- или в `~/.local/` / `/tmp/` — вне репозитория

**Уровень дистрибутива:**
- CI/пре-релизный чек на паттерны runtime-файлов (`*-YYYY*.log`, UUID-имена)
- `git ls-files | grep -E '[0-9]{8}'` → артефакты с датой в имени = подозрение

## Связи

- Дополняет: [DP.M.399](../03-methods/DP.M.399-diagnose-empty-dir-by-mtime-before-delete.md) (диагностика директории по mtime до удаления)
- Смежно: [DP.FM.289](DP.FM.289-orphan-file-upstream-merge-manifest-gap.md) (orphan-файл из upstream merge)