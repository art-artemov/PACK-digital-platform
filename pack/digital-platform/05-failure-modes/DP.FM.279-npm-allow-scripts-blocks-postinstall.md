---
id: DP.FM.279
type: failure-mode
status: draft
created: 2026-07-14
trust:
  F: 2
  G: domain
  R: 0.3
epistemic_stage: emerging
---

# FM: npm allow-scripts блокирует postinstall при установке инструментов с нативными бинарниками

## 1. Паттерн ошибки

npm-настройка `allow-scripts` (глобальная или проектная) блокирует postinstall-скрипты. Инструменты с нативными бинарниками — пример: Claude Code (`@anthropic-ai/claude-code`) — требуют postinstall для загрузки платформо-зависимой optional-зависимости. Без него нативный бинарник не устанавливается.

**Симптом:** CLI падает с `Error: claude native binary not installed`; IDE-расширения показывают «MCP controls aren't available».

## 2. Почему это ошибка

`allow-scripts=false` — защитная мера npm против произвольного выполнения кода при установке. Но обобщённый запрет ломает легитимные postinstall'ы инструментов, которым нужна платформо-специфичная загрузка или компиляция. Пакет устанавливается без ошибок, поэтому ошибка обнаруживается только при запуске.

## 3. Антипаттерн → Паттерн

| Антипаттерн | Паттерн |
|-------------|---------|
| Глобальный `allow-scripts=false` без исключений | Точечное исключение для доверенных пакетов |
| Диагностика начинается с переустановки без анализа настроек npm | Сначала `npm config get allow-scripts` — найти корень |
| Ручной postinstall как разовый фикс | `npm config set allow-scripts=<package> --location=user` — постоянный фикс |

**Как чинить (проверено Claude Code v2.1.175, 2026-07-11):**
1. `npm install -g @anthropic-ai/claude-code` — докачивает optional-зависимость
2. `node <install-path>/claude-code/install.cjs` — postinstall вручную
3. `npm config set allow-scripts=@anthropic-ai/claude-code --location=user` — постоянный фикс (иначе ломается при каждом `npm update -g`)

## 4. Тест обнаружения

- `npm config get allow-scripts` → возвращает `false` или не возвращает имя нужного пакета?
- `npm install -g <пакет>` → нет postinstall-вывода в логе?
- `ls $(npm root -g)/@anthropic-ai/claude-code-darwin-arm64` → директория отсутствует?

## 5. Связанные документы

- IWE: `memory/lessons_claude-cli-mcp.md §1` — конкретная инструкция для Claude Code (IWE-уровень)