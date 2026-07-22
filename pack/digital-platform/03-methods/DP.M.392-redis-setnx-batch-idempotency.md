---
id: DP.M.392
name: "Идемпотентность батча через Redis SETNX с TTL"
type: method
domain: digital-platform / reliability
status: active
valid_from: 2026-07-12
sources:
  - "session-close 2026-07-12; WP-8 (мастер-план push-сервиса)"
related:
  see_also: [DP.M.388, DP.M.329]
tags: [redis, idempotency, setnx, ttl, queue, batch]
---

# DP.M.392 — Идемпотентность батча через Redis SETNX с TTL

## Определение

Механизм гарантии однократной обработки батча в очереди задач через атомарную операцию Redis SETNX с TTL.

## Проблема

При сбое delivery-воркера и повторной отправке батча диспетчером воркер может обработать его дважды — дублирование уведомлений, двойное списание, повторные side-эффекты.

## IPO

| Элемент | Описание |
|---------|----------|
| Входы | batchId из запроса; TTL — ожидаемый максимальный window ретраев диспетчера |
| Обработка | `SET batch:{batchId} 1 NX EX {ttl_seconds}` — атомарно, только если ключа нет |
| Выходы | OK → батч новый, обрабатывать; nil → батч уже обработан, вернуть 200 без действий |

## Шаги метода

1. При получении батча выполнить `SET batch:{batchId} 1 NX EX {ttl}`.
2. Если SET вернул nil (ключ уже существует) → вернуть 200 без дублирования.
3. Если SET вернул OK → продолжить обработку батча.
4. TTL выбирается как «максимальный разумный window retry диспетчера» (обычно 24–48 ч).

## Место среди механизмов идемпотентности Pack

- [DP.M.388](DP.M.388-idempotent-batches-resumability-cursor.md) — control-plane сторона (курсор + генерация batchId); DP.M.392 — delivery-plane сторона (проверка того же batchId через Redis)
- [DP.M.329](DP.M.329-webhook-idempotency-db-constraint.md) — тот же принцип (атомарная проверка «уже видел?»), но constraint на уровне БД, не Redis-ключ с TTL — выбор зависит от того, нужен ли автоматический TTL-based expiry или постоянная запись

## Применимость

Переносимый паттерн надёжности для любой очереди задач с Redis (BullMQ, Sidekiq, Celery). Не привязан к FCM или push-домену — применим к email, SMS, webhooks, import jobs.