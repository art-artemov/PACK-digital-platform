---
id: DP.M.387
name: "Opt-out хранение подписок: инверсия таблицы (нет строки = подписан)"
type: method
domain: digital-platform / data-design
status: active
valid_from: 2026-07-11
sources:
  - "session-close 2026-07-11; WP-7 (мастер-план push-сервиса, ArchGate)"
related:
  complements: [DP.M.388]
tags: [data-design, opt-out, subscriptions, storage-inversion, scale]
---

# DP.M.387 — Opt-out хранение подписок: инверсия таблицы

## Определение

Паттерн хранения пользовательских подписок для opt-out-семантики: в таблице хранятся **только отписки**, отсутствие строки означает согласие по умолчанию.

## IPO

- **Вход:** Аудитория N пользователей × M типов подписок; семантика по умолчанию — «подписан»
- **Процесс:** Хранить только явные отписки. Проверка: `NOT EXISTS (SELECT 1 WHERE userId=? AND typeId=?)`
- **Выход:** Таблица O(n_unsubscribers × m) вместо O(N × M)

## Условие применения

**Только для opt-out-семантики.** При opt-in (требуется явное согласие) — инверсия неверна: отсутствие строки = «не давал согласие», а не «согласен».

## Пример (push-подписки)

```sql
CREATE TABLE hd_push_subscriptions (
  user_id     INT          NOT NULL,
  type_id     VARCHAR(32)  NOT NULL,
  device_id   VARCHAR(64)  NULL,       -- задел под per-device (пока NULL)
  unsubscribed_at TIMESTAMP NOT NULL,
  PRIMARY KEY (user_id, type_id)
);

-- Проверка подписки: TRUE если строки НЕТ
SELECT NOT EXISTS(
  SELECT 1 FROM hd_push_subscriptions
  WHERE user_id = ? AND type_id = ?
) AS is_subscribed;
```

## Преимущества

1. **Масштаб:** 150k аудитория × N типов → таблица остаётся малой (только отписавшиеся)
2. **Без backfill:** Новый тип подписки = строки не нужны, все автоматически подписаны
3. **deviceId задел:** При переходе на per-device управление — добавить NOT NULL и индекс

## Аналогии

Переносимо на: feature flags по умолчанию включены, email-уведомления с opt-out, marketing preferences.

## Ограничение

Не применять для типов `required: true` (транзакционные уведомления без права отписки). API должен возвращать 422 при попытке отписаться от required-типа — это проверка бизнес-правила, не отсутствия строки.