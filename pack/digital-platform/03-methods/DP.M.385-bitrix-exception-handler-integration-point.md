---
id: DP.M.385
name: "Интеграция error-tracking в Bitrix через exception_handling, а не глобальные хендлеры SDK"
type: method
domain: digital-platform / legacy-integration
status: active
valid_from: 2026-07-15
sources:
  - "session-close 2026-07-11 (сессия WP-6, интеграция GlitchTip в Humdes web)"
related:
  complements: [DP.M.386]
tags: [bitrix, error-tracking, legacy-cms, exception-handling, sentry, glitchtip]
---

# DP.M.385 — Интеграция error-tracking в Bitrix через exception_handling

## Определение

Подключение error-tracking SDK (Sentry/GlitchTip) к Bitrix-приложению через штатную точку расширения `exception_handling → log → class_name`, а не через глобальные error-хендлеры SDK. Это предотвращает конфликты порядка с хендлерами Bitrix и дублирование событий.

## IPO

- **Вход:** Bitrix-приложение без централизованного error-tracking
- **Процесс:** Создать наследник `Bitrix\Main\Diag\ExceptionHandlerLog`, зарегистрировать в `exception_handling.log.class_name`
- **Выход:** Все необработанные исключения, PHP-ошибки по маске и фаталы (включая cron) → error-tracking

## Метод

1. Создать класс-наследник `Bitrix\Main\Diag\ExceptionHandlerLog`
2. Переопределить метод приёма ошибок → вызов `Sentry\captureException()` / `Sentry\captureMessage()`
3. В `.settings.php`: `'exception_handling' => ['log' => ['class_name' => MyErrorTrackingLog::class]]`
4. Ленивая инициализация SDK: SDK инициализируется при первой ошибке (не при каждом хите)
5. Отключить собственные error-хендлеры SDK: Bitrix управляет порядком, SDK дублирует
6. PII-скраб: `send_default_pii=false`, маскировать чувствительные query-параметры

## Ключевые свойства

- **Cron coverage**: CLI подключает пролог Bitrix → хендлер активен и в cron-задачах
- **No overhead**: SDK поднимается ленивой инициализацией, обычные хиты без накладных расходов
- **No duplicates**: SDK не устанавливает свои error-хендлеры → Bitrix управляет порядком

## Общий принцип

Интегрироваться в штатную точку расширения платформы, а не поверх неё. Применимо к любому сервису наблюдаемости (APM, tracing) в legacy CMS с собственным error-хендлером.

## Различение

Этот метод — точка подключения (где). DP.M.386 — стратегия перехода от файловых логов к error-tracking (когда и как мигрировать).

## Источник

WP-6 (интеграция GlitchTip в Humdes web): подтверждено на практике, включая cron-задачи.