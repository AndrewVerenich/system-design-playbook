# Раунд 02 — Review (финальный)

> Reviewer: interviewer (AI)  
> Дата: 2026-09-16  
> Документ: [`DESIGN.md`](./DESIGN.md) + [`diagrams/high-level.jpg`](./diagrams/high-level.jpg)

---

## Вердикт

**Strong Hire** для задачи Webhook Dispatcher (mid+/senior).

Сильный, связный дизайн: быстрый accept, durable outbox через CDC, fan-out в отдельный `topic.notification`, независимый retry/DLQ, HMAC + idempotency, per-tenant изоляция.  
Заметно лучше «наивного» варианта (синхронный POST на URL в том же запросе, что accept). Глубина fan-out / ordering / dual-write — senior-level.

Чуть тяжелее минимально необходимого для 200 events/s (shard+topic per tenant, полный CDC-pipeline) — но trade-off осознанный и защищаемый на интервью.

---

## Strengths

1. **Скоуп чистый:** dispatcher без доменной бизнес-логики; триггер снаружи.
2. **Accept ≠ delivery:** `202` после INSERT; чужой HTTP не в критическом пути.
3. **Dual-write закрыт прагматично:** домен ретраит до `202` + идемпотентный `eventId` (не притворяемся, что HTTP ingress = transactional outbox домена).
4. **Pipeline зрелый:** MySQL → Debezium → `topic.event` → **event-processor** → N× `topic.notification` → http worker → DLQ / status.
5. **Fan-out правильный:** одно сообщение на подписчика → независимые retry/scale.
6. **Ordering осознан:** `partitionKey` на бизнес-сущности; в notification — `partitionKey:webhookId`.
7. **Auth разделён:** `apiKey` inbound vs `signingSecret` outbound HMAC.
8. **Retry policy конкретная:** 1m/5m/30m; 5xx/timeout vs 400/410 → DLQ.
9. **At-least-once + ключи:** accept / attempts / `X-Idempotency-Key`.
10. **NFR с цифрами** + capacity (ingress → outbound × fan-out, retention).
11. **Схема** читается end-to-end; есть registration, cleanup, status update, DLQ.
12. Эволюция дизайна в процессе (partitionKey, fan-out через Kafka, secret не в plaintext) — хороший сигнал для интервью.

---

## Gaps / risks (не валят дизайн)

1. **Secret на схеме vs в тексте.** На jpg у processor всё ещё читается как enrich «webhooks **and secret**». В `DESIGN.md` зафиксировано: **secret lookup в http worker**, не plaintext в Kafka. Подровнять подпись на схеме → только `webhookId` (+ url), secret у worker.
2. **Per-tenant MySQL + Kafka topics** при 200 eps — ops-heavy. На собесе держи фразу: «v1 изоляция; при тысячах tenants → shared topic + `tenant_id`».
3. **`delivery_attempts` на схеме** не отдельным боксом (есть update status) — для API «статус доставки» ок, но модель стоит держать в голове/доке (уже в §7).
4. **Observability** слабо: CDC lag, fan-out lag, delivery success rate, DLQ depth, accept p99 — must в проде, на интервью хватит 1 предложения.
5. **SSR/webhook SSRF:** outbound на произвольный URL — block private IP / allowlist (мелочь security).
6. **Ротация secret / смена URL:** re-fetch перед POST упомянут в доке — хорошо; на схеме не обязателен.
7. Статус документа был `draft` → после review → **`reviewed`**.

---

## Follow-up questions (как добивали бы на собесе)

### 1. Lag CDC / event-processor (например 10 минут)

**Вопрос:** Event-processor / CDC отстал на 10 минут. Где копятся данные? Что видит домен? Когда догоните?

**Ответ:** Accept уже успешен — событие в MySQL (`events`), домен получил `202`. Доставка eventual: клиенты получат webhook позже.

Где копятся данные:
- CDC не читает → давление на WAL/binlog / replication slot;
- CDC ок, processor отстаёт → lag `topic.event`;
- workers медленные → lag `topic.notification`.

Действия: мониторинг accept success, CDC lag, consumer lag, DLQ depth; scale bottleneck; не теряем данные — source of truth MySQL до cleanup.

**Фраза:** lag бьёт по SLO *доставки*, не по durability accept.

### 2. Один `eventType` × 10k webhooks

**Вопрос:** Один tenant подписал 10k webhooks на `order.paid`. Что будет с fan-out и как ограничишь?

**Ответ:** Наивный fan-out даст 10k produce + 10k HTTP на один event.

Защита:
1. Product limits — max webhooks на tenant / на `eventType`;
2. Backpressure в event-processor (limit concurrent produce);
3. Slow lane / отдельный пул для огромного fan-out;
4. Batch produce в `topic.notification`;
5. На экстремуме — один URL у клиента (их внутренняя шина), а не 10k наших HTTP.

**Фраза:** деградация через лимиты подписок важнее героического fan-out.

### 3. Глобальный порядок всех событий tenant

**Вопрос:** Клиент требует порядок *всех* событий tenant, не только по `orderId`. Что скажешь?

**Ответ:** Обещаем порядок по **`partitionKey` (агрегат)** и по **`partitionKey:webhookId`** в notification.

Глобальный порядок на весь tenant ≈ одна partition → нет параллелизма, один медленный путь тормозит всё.

Обычно бизнесу нужен порядок по заказу/пользователю, не total order компании. Если must — single-partition + честный trade-off throughput, либо клиент упорядочивает по `occurredAt`/`sequence`.

**Фраза:** global total order и горизонтальный scale почти несовместимы — уточняем реальный requirement.

### 4. Ротация `signingSecret` без даунтайма

**Вопрос:** Нужно ротировать `signingSecret` без даунтайма доставки. Как?

**Ответ:**
1. `POST /webhooks/{id}/rotate-secret` → новый secret; старый valid ещё grace period (dual-secret);
2. Клиент обновляет проверку (принимает оба в окне);
3. Worker при отправке делает **lookup secret by webhookId** → подписывает current (не secret из Kafka);
4. После grace — invalidate previous.

Аргумент не класть plaintext secret в Kafka: in-flight сообщения не содержат секрет в брокере.

**Фраза:** ротация = dual-secret window + lookup на send, не restart workers.

### 5. CDC vs прямой produce в Kafka из event-gateway

**Вопрос:** Зачем CDC, если event-gateway может писать в Kafka сразу после INSERT?

| | CDC (выбрано) | Прямой produce после INSERT |
|---|---|---|
| Dual-write DB↔Kafka | нет | риск «DB ok, Kafka fail» |
| Replay | из MySQL / snapshot | сложнее |
| Latency доставки | + CDC lag | ниже |
| Ops | Debezium, slots | проще стек |

Прямой produce ок на малом масштабе + outbox-poller. CDC оправдан: «INSERT в events ⇒ гарантированно уедет в Kafka» без второго write в API-пути.

**Фраза:** CDC — outbox без publisher-кода в gateway; платим ops и lag за отсутствие dual-write.

---

## Оценка по рубрике интервью

| Критерий | Оценка |
|----------|--------|
| Clarifying / scope | Strong |
| NFR + capacity | Strong |
| High-level design | Strong |
| Deep dive (fan-out, ordering, idempotency) | Strong |
| Reliability / retry / DLQ | Strong |
| Security (apiKey / HMAC / secret) | Good → Strong (после фикса secret∉Kafka) |
| Trade-offs / overengineering | Good (тяжёлый tenant isolation — осознанно) |
| Communication | Strong |

**Итог:** Strong Hire на Webhook Dispatcher.

---

## Что сделать после review (опционально)

- [ ] На jpg: убрать «secret» из enrich processor; подпись «secret lookup в http worker»
- [ ] Статус в `DESIGN.md` → `accepted` (после мелких правок схемы)
- [ ] §12: своими словами 1–2 follow-up
- [ ] ADR по желанию: `adr/0001-fanout-via-notification-topic.md`, `adr/0002-no-secret-in-kafka.md`
