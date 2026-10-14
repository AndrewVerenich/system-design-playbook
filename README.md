# System Design Playbook

Коллекция архитектурных разборов крупных систем в формате System Design interview.  
Репозиторий демонстрирует практику проектирования: требования, оценка нагрузки, high-level архитектура, модель данных, API, масштабирование, отказоустойчивость и явные trade-offs.

Каждый раунд — законченный design-документ (ADR/RFC-стиль), пригодный и для подготовки к интервью, и как материал портфолио.

## Содержание: раунды

Краткий список — ссылки ведут к полному описанию в [разделе ниже](#rounds-overview).

1. [pastebin](#1-pastebin) — текстовые paste, TTL, read/write path  
2. [webhook-dispatcher](#2-webhook-dispatcher) — доставка webhooks, retry, async fan-out  
3. [rate-limiter](#3-rate-limiter) — distributed rate limiting **← текущий раунд**  
4. *(идеи: [payment system](#идеи-следующих-раундов), [job scheduler](#идеи-следующих-раундов), …)*

---

## Как проходить раунд

1. **Brief** — задача и ограничения (как на интервью).
2. **Draft** — свой `DESIGN.md` + схема в `diagrams/`.
3. **Review** — разбор: сильные стороны, дыры, follow-up вопросы.
4. **Revise** — правка до accepted-версии.

Шаблон: [`_template/DESIGN.md`](./_template/DESIGN.md).

---

<a id="rounds-overview"></a>

## Раунды

### 1. [pastebin](./rounds/01-pastebin/BRIEF.md)

Проектирование Pastebin-like сервиса: сохранение текста, выдача по короткому id, TTL, опционально burn-after-read.

**Описание:**
- Простая система для отработки каркаса интервью end-to-end.
- Фокус: где хранить тело paste, expiry, id generation, cache, read-your-writes.

**Артефакты:**
- [BRIEF.md](./rounds/01-pastebin/BRIEF.md) — условие раунда
- [DESIGN.md](./rounds/01-pastebin/DESIGN.md) — дизайн
- [REVIEW.md](./rounds/01-pastebin/REVIEW.md) — review
- [diagrams/](./rounds/01-pastebin/diagrams/) — схемы

**Фокус компетенций:**
- Interview structure end-to-end
- Blob vs DB storage trade-off
- TTL / expiration
- Concurrent read semantics

---

### 2. [webhook-dispatcher](./rounds/02-webhook-dispatcher/BRIEF.md)

Сервис доставки webhooks: регистрация endpoint’ов, приём событий, async fan-out, retry при сбоях внешнего HTTP.

**Описание:**
- Accept: `POST /internal/events` → MySQL → `202` (не ждём чужой HTTP).
- Pipeline: CDC → `topic.event` → event-processor (fan-out) → N× `topic.notification` → http worker.
- Outbound: HMAC, idempotency, retry/DLQ; регистрация webhooks по apiKey.

**Артефакты:**
- [BRIEF.md](./rounds/02-webhook-dispatcher/BRIEF.md) — условие раунда
- [DESIGN.md](./rounds/02-webhook-dispatcher/DESIGN.md) — дизайн
- [REVIEW.md](./rounds/02-webhook-dispatcher/REVIEW.md) — review
- [diagrams/](./rounds/02-webhook-dispatcher/diagrams/) — схемы

**Фокус компетенций:**
- Async processing / outbox + CDC
- Fan-out (1 event → N notifications)
- Retry / backoff / timeouts / DLQ
- At-least-once + idempotency
- Failure isolation

---

### 3. [rate-limiter](./rounds/03-rate-limiter/BRIEF.md)

Distributed rate limiter для API Gateway / сервисов: allow/deny по ключу (user, IP, apiKey) с окном и лимитом.

**Описание:**
- Hot-path decide, алгоритмы (token/leaky bucket, fixed/sliding window), Redis, multi-instance.
- Фокус: latency, atomicity, hot keys, fail-open vs fail-closed.

**Артефакты:**
- [BRIEF.md](./rounds/03-rate-limiter/BRIEF.md) — условие раунда
- [DESIGN.md](./rounds/03-rate-limiter/DESIGN.md) — дизайн
- [REVIEW.md](./rounds/03-rate-limiter/REVIEW.md) — review
- [diagrams/](./rounds/03-rate-limiter/diagrams/) — схемы

**Статус:** brief ready — обсуждение / дизайн

**Фокус компетенций:**
- Rate limiting algorithms
- Distributed counters (Redis)
- Hot path latency
- Failure modes (Redis down)

---

## Идеи следующих раундов

Кандидаты после Rate Limiter (ещё без BRIEF — выберем и заведём папку):

| # | Система | Фокус |
|---|---------|--------|
| 4 | **Payment System** | идемпотентность платежей, ledger, exactly-once эффекты, reconciliation, PSP/webhooks, consistency vs latency |
| 5 | **Job Scheduler** | cron/delayed jobs, distributed locks, at-least-once execution, retry/backoff, partitioning очереди, clock skew |

Оба хорошо стыкуются с уже пройденным: webhooks → payment callbacks; rate limiter / Redis → scheduler locks; у тебя в platform уже есть `distributed-task-scheduler` и banking/saga-кейсы.

---

## Диаграммы

Схемы — в `rounds/<round>/diagrams/` и вставляются в `DESIGN.md` соответствующего раунда.
