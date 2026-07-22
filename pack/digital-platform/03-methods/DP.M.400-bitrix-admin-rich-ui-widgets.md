---
id: DP.M.400
name: Встроенные rich-виджеты Bitrix для custom admin-форм
type: method
domain: digital-platform / bitrix / admin-ui
status: draft
valid_from: 2026-07-17
sources:
  - session-close 2026-07-16, WP-26 Ф8 (DS-humdes/web, commit 87d0d9a0)
related:
  complements: [DP.M.401]
tags: [bitrix, admin, colorpicker, html-editor, rich-ui, custom-module, php]
---

# DP.M.400 — Встроенные rich-виджеты Bitrix для custom admin-форм

## Суть метода

При создании кастомных admin-форм в Bitrix использовать **штатные UI-компоненты платформы** вместо собственного HTML:
- `bitrix:main.colorpicker` — выбор цвета с превью-квадратом
- `CFileMan::AddHTMLEditorFrame()` — WYSIWYG HTML-редактор

Повторяющиеся поля (например, несколько цветовых пикеров) выносить в хелпер-функцию (принцип P2).

## Компоненты

### Colorpicker
```php
// Подключение
$APPLICATION->IncludeComponent('bitrix:main.colorpicker', '', [
    'inputName' => 'UF_COLOR',
    'inputValue' => htmlspecialchars($currentValue),
    'inputId' => 'color-input-id',
]);
```
Образец из ядра: `sale/admin/status_edit.php`.

### HTML-редактор
```php
// Требует модуль fileman
\Bitrix\Main\Loader::includeModule('fileman');
CFileMan::AddHTMLEditorFrame(
    'UF_HTML_TEXT',           // name
    htmlspecialchars($value), // content
    'UF_HTML_TEXT_type',      // type field name
    'html',                   // forced mode: html
    [],                       // settings
    500,                      // height
    '100%',                   // width
    '',                       // id
    ''                        // title
);
```
Параметр `'html'` фиксирует режим (без переключателя текст/HTML).

### Хелпер для повторяющихся полей
```php
function hdcStoriesRenderColorField(string $name, string $value, string $label): void
{
    echo '<tr><td>' . htmlspecialchars($label) . '</td><td>';
    global $APPLICATION;
    $APPLICATION->IncludeComponent('bitrix:main.colorpicker', '', [
        'inputName' => $name,
        'inputValue' => htmlspecialchars($value),
    ]);
    echo '</td></tr>';
}
```

## Правило

**Перед написанием кастомного HTML для поля** — проверить, есть ли штатный компонент Bitrix. Ядро покрывает: цвет, WYSIWYG, файл, дата, иконка.

## Тест

«Написан ли кастомный HTML-пикер для цвета или редактора, когда есть `bitrix:main.colorpicker` / `CFileMan::AddHTMLEditorFrame`?» Да → метод не применён.

## Инцидент

WP-26 Ф8 (2026-07-16): поля «Фон», «Кнопка — фон», «Кнопка — цвет текста» — colorpicker, «HTML-текст» — HTMLEditorFrame. Деплой прошёл без переустановки модуля (прокси-стаб Ф5) — новые admin-элементы не требуют переустановки, только изменение схемы (см. [DP.M.401](DP.M.401-bitrix-ensurecolumn-reinstall-required.md)).