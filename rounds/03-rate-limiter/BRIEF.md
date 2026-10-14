# Раунд 03 — Rate Limiter

**Роль AI:** интервьюер + reviewer.  
**Твой deliverable:** `DESIGN.md` + схема в `diagrams/`.  
**Время:** не ограничено; можно идти по шагам в чате.  
**Предыдущие раунды:** [Pastebin](../01-pastebin/), [Webhook Dispatcher](../02-webhook-dispatcher/).

---

## Задача

Спроектировать **сервис ограничения частоты запросов** (distributed rate limiter), которым пользуются API Gateway / BFF / микросервисы.

Пример: «не больше **100 запросов в минуту** на `apiKey` / `userId` / IP».

Клиент (или gateway) перед обработкой бизнеса спрашивает: *можно ли пропустить этот запрос?* — или лимитер стоит как sidecar/middleware.

---

## Функциональные требования (стартовый скоуп — уточним вместе)

**Must have (ориентир)**

- Проверить лимит: `allow` / `deny` (+ ideally remaining, reset/retry-after)
- Конфиг правила: key dimension + limit + window (например 100/min)
- Работать при **нескольких инстансах** приложения (не только in-process)
- Разумное поведение при отказе хранилища лимитера (fail-open vs fail-closed — обсудим)

**Nice to have**

- Несколько правил на один запрос (IP + user + endpoint)
- Soft limit / soft throttle (не hard deny)
- Админ API CRUD правил
- Метрики: denied rate, latency decide

**Вне скоупа v1 (пока)**

- Полноценный WAF / bot detection
- Billing / quotas с деньгами
- UI дашборд
- Per-tenant биллинг-планы как продукт

---

## Нефункциональные цели (ориентир)

| NFR | Ориентир |
|-----|----------|
| Decide latency | p99 **&lt; 5–10 ms** (hot path) |
| Throughput | peak **~50k decide/s** (можно уточнить) |
| Availability | высокая; лимитер не должен стать SPOF без политики |
| Accuracy | «почти точный» ок; обсудим погрешность sliding vs fixed window |
| Consistency | под нагрузкой допустима небольшая погрешность ради latency |

Ключевая идея раунда: **быстрое решение allow/deny** в distributed-среде + выбор алгоритма и хранилища.

---

## На что обратить внимание (без готового решения)

1. Алгоритм: Token Bucket / Leaky Bucket / Fixed Window / Sliding Window  
2. Где state: Redis vs local + sync  
3. Atomicity: `INCR` + TTL, Lua, Redlock?  
4. Hot key (один vip user / один IP)  
5. Fail-open vs fail-closed при падении Redis  
6. Куда встроить: library vs sidecar vs центральный service  

Не усложняй Kafka/CDC без необходимости — для классического rate limiter часто хватает Redis.

---

## Что сдать

1. Заполни [`DESIGN.md`](./DESIGN.md) (можно по секциям в чате).  
2. Схема → `diagrams/rate-limiter.jpg`.  
3. Напиши «Раунд 03 готов к review» или продолжай обсуждение с FR.

---

## Зачем эта задача сейчас

После Pastebin (KV) и Webhook (async fan-out) — другая ось: **корректность и latency на горячем пути**, distributed counters, алгоритмы. Классика Senior system design interview.
