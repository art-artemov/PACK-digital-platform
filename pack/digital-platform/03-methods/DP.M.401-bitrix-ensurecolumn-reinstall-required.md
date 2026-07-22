---
id: DP.M.401
name: Bitrix ensureColumn — последовательность деплоя схемы через переустановку модуля
type: method
domain: digital-platform / bitrix / schema-deployment
status: draft
valid_from: 2026-07-17
sources:
  - session-close 2026-07-16, WP-26 Ф6 (DS-humdes/web, сессия 16 июля)
related:
  complements:
    - DP.M.400  # bitrix admin rich UI widgets (та же сессия)
tags: [bitrix, schema, migration, ensurecolumn, module-install, deployment, orm, datamanager]
---

# DP.M.401 — Bitrix: ensureColumn требует переустановки модуля, не git push

## Суть метода

В кастомных Bitrix-модулях изменение схемы таблицы через `ensureColumn()` в методе `installDB()` **выполняется только при (пере)установке модуля** — не при `git push` и не при перезагрузке страницы.

**Правильная последовательность деплоя схемы:**

1. `git push` — код с новым полем в `getMap()` + `ensureColumn()` в `installDB()`
2. **Немедленно** переустановить модуль: Настройки → Модули → «Переустановить»

Промежуток между шагами 1 и 2 — **реальное окно поломки на prod** (код обращается к несуществующей колонке).

## Механизм

`ensureColumn()` внутри — это `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`. Bitrix вызывает `installDB()` только при событии установки модуля, поэтому:

```php
// В TableSchema::getMap() — новое поле
new IntegerField('SORT_ORDER', ['default_value' => 0]),

// В installDB() — применение ALTER
TableSchema::getEntity()->getConnection()->queryExecute(
    "ALTER TABLE hd_stories_slides ADD COLUMN IF NOT EXISTS SORT_ORDER INT DEFAULT 0"
);
// или через ORM helper: ensureColumn()
```

Без переустановки: `getMap()` знает о поле → ORM пытается SELECT/INSERT → `Unknown column 'SORT_ORDER'`.

## Важное различение

`ensureColumn` ≠ migration runner (как Laravel Artisan или Flyway). Это `ALTER TABLE IF NOT EXISTS`, выполняемый вручную при install. Нет автоматики при деплое.

## Для NULL-совместимых полей

Если новое поле nullable с DEFAULT — код **не упадёт** до переустановки при чтении (NULL вернётся), но поле будет недоступно для записи. Окно поломки тихое — симптом появится не сразу.

## Тест

«Код видит поле в `getMap()`, а в БД колонки нет?» → Переустановка модуля не была выполнена после деплоя.

## Инцидент

WP-26 Ф6 (2026-07-16): добавление поля `NAME` в `StoryTable::getMap()` + `ensureColumn` в `installDB()`. После git push поле недоступно — разблокировано переустановкой модуля (Ф7). Деплой Ф8 (новые admin-элементы, [DP.M.400](DP.M.400-bitrix-admin-rich-ui-widgets.md)) прошёл без переустановки — подтверждает: только схема требует переустановки.