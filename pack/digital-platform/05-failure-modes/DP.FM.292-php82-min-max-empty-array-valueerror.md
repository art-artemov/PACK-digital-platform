---
id: DP.FM.292
name: "PHP 8.2+: min([])/max([]) бросает ValueError вместо false + warning"
type: failure-mode
domain: DP
status: active
valid_from: 2026-07-17
source: "session-close 2026-07-16; WP-26 Ф3 (DS-humdes/web, commit df007267)"
tags: [php, php82, min, max, valueerror, migration, legacy, bitrix]
---

# DP.FM.292 — PHP 8.2+: min([])/max([]) бросает ValueError на пустом массиве

## Паттерн

В PHP 8.2+ функции `min([])` и `max([])`, вызванные на **пустом массиве**, бросают `ValueError`:

```
ValueError: max(): Argument #1 ($value) must contain at least one element
```

В PHP 7 то же поведение возвращало `false` с `E_WARNING` — код, написанный под PHP 7 с проверкой `if ($val === false)`, тихо работает пока массив непуст.

## Симптом

Падение появляется **только на граничном случае** — первой записи в пустой коллекции, нулевом слайде, новом объекте без данных. На уже заполненных данных код работает нормально.

Часто маскируется под «работало, вдруг сломалось» — потому что баг воспроизводится только при создании нового объекта.

## Диагностика

**Тест:** «Ошибка воспроизводится только при пустой коллекции (первый элемент)?» Да → PHP 8.2+ `min`/`max` failure mode.

Проверить версию PHP: `php -v` или `phpinfo()`. Если >= 8.2 и вызывается `min()`/`max()` без проверки — уязвимость есть.

## Fix

```php
// До (PHP 7 — опасно на PHP 8.2+)
$maxOrder = max($orders); // ValueError если $orders = []

// После
$maxOrder = empty($orders) ? 0 : max($orders);
```

Или через явный guard с null-семантикой:

```php
$maxOrder = count($orders) > 0 ? max($orders) : null;
```

## Применимость

Любой PHP 7 → PHP 8.2+ migration в legacy-коде (Bitrix, Laravel 8, WordPress), где `min`/`max` вызываются на переменных, теоретически способных оказаться пустым массивом.

## Инцидент

WP-26 Ф3 (2026-07-16): `ValueError` в `hdc.stories.slide.edit.php:213` при попытке сохранить первый слайд истории (массив `$orders` пуст при отсутствии слайдов). Проект на PHP 8.3 (`web/CLAUDE.md`), код унаследован из PHP 7 эпохи.