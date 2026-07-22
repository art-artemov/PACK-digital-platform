---
id: DP.M.388
name: "Идемпотентные батчи с курсором возобновляемости для массовой рассылки"
type: method
domain: digital-platform / reliability
status: active
valid_from: 2026-07-11
sources:
  - "session-close 2026-07-11; WP-7 (мастер-план push-сервиса, ArchGate)"
related:
  complements: [DP.M.387]
tags: [reliability, idempotency, cursor, batch, exactly-once, mass-delivery]
---

# DP.M.388 — Идемпотентные батчи с курсором возобновляемости

## Определение

Контракт надёжности между control plane и delivery plane при массовой рассылке: диспетчер разбивает аудиторию на **идемпотентные батчи фиксированного размера**, кампания хранит **курсор прогресса**. Вместе они дают exactly-once delivery при сбоях.

## IPO

- **Вход:** Кампания рассылки (N получателей); delivery plane — отдельный сервис
- **Процесс:**
  1. Диспетчер нарезает аудиторию на батчи по BATCH_SIZE с уникальным `batchId = campaignId:cursor`
  2. Кампания сохраняет курсор после каждого успешного батча
  3. Delivery plane проверяет идемпотентность по `batchId` (Redis SET NX + TTL)
- **Выход:** Каждый получатель обработан ровно один раз, даже при падении диспетчера или delivery

## Гарантии

| Сценарий | Результат |
|----------|-----------|
| Падение диспетчера после K батчей | Возобновление с K+1 (курсор) |
| Повторная отправка того же батча | Delivery пропускает по idempotency key |
| Временный сбой delivery service | Батч повторится после восстановления |

**Нет дублей (идемпотентность) + Нет потери хвоста (курсор) = exactly-once delivery**

## Пример (control plane)

```typescript
const BATCH_SIZE = 1000;

async function dispatchCampaign(campaignId: string) {
  let cursor = await getCursor(campaignId); // 0 при первом запуске

  while (true) {
    const batch = await getRecipients(campaignId, cursor, BATCH_SIZE);
    if (batch.length === 0) {
      await markCampaignDone(campaignId);
      break;
    }

    const batchId = `${campaignId}:${cursor}`;
    await sendBatch({ batchId, tokens: batch });   // idempotent endpoint
    await saveCursor(campaignId, cursor + batch.length);
    cursor += batch.length;
  }
}
```

## Применимость

Массовые outbound-операции: push-уведомления, email-рассылки, SMS, webhooks к N получателям.

Паттерн ортогонален механизму идемпотентности на delivery-стороне (Redis SET NX с TTL — отдельный метод).