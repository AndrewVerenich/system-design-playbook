# Система: Rate Limiter

> Раунд: 03  
> Автор: Andrey Verenich  
> Статус: reviewed  

> Timebox: без жёсткого лимита

## 1. Уточняющие вопросы

Что спросил у интервьюера / какие assumptions зафиксировал:

- Лимитер нужен для **API Gateway / BFF / сервисов** в multi-instance среде (не только in-process).
- Решение **allow/deny** на горячем пути; бизнес-логика за лимитером.
- **Ключ лимита (зафиксировано):** dimension из правила — `USER_ID` | `API_KEY` | `IP` (+ опционально match по `METHOD:path`). Несколько matched rules → **AND** (любой deny = 429).
- Точность «почти exact» допустима ради latency; идеальный global exact не обязателен.
- **Local safety net (зафиксировано):** при fail-open Redis — грубый **in-process** token bucket на инстанс (без координации), чтобы один pod не ушёл в бесконечный over-allow.
- **Admin CRUD правил:** простой Admin API / ops SQL в MySQL; UI вне скоупа. Метрики allow/deny/latency/store_errors — must для ops.
- **Вводная интервьюера:** при недоступности store (Redis) — **fail-open** (пропускаем запрос). Кратковременный over-allow лучше полного outage API. Алерт обязателен; опционально local fallback-лимит на инстанс. Чувствительные endpoint’ы с fail-closed — nice-to-have, не must v1.
- **Доставка правил (зафиксировано):** MySQL = SoT; **CDC → Kafka** пушит изменения; **каждый** gateway-инстанс — свой consumer (своя копия событий) → in-memory snapshot. Decide path **только RAM + Redis**. Bootstrap: полный snapshot из MySQL (или compacted topic). Kafka/CDC lag → last snapshot + `rules_stale`. Опционально редкий reconcile-poll как safety net.
- **Лимитер:** library, встроенная в gateway.
- **Counters:** Redis Cluster (shard by key), HA через replicas (failover); decide/writes — на primary шарда.
- **Алгоритм:** Token Bucket; атомарность через **Lua** на Redis.
- **Ответ клиенту:** 429 + `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `Retry-After` при deny.

## 2. Функциональные требования

**Must have (v1)**

1. **Decide:** по запросу с ключом (`subject` + опционально `resource`/`endpoint`) вернуть **allow** или **deny**.
2. В ответе: **remaining**, **limit**, **reset/retry-after** (чтобы клиент/gateway мог выставить заголовки).
3. **Конфиг правила:** `limit` + `window` (например 100 / 60s) + тип ключа.
4. Корректная работа при **нескольких инстансах** приложения (общий state).
5. Идемпотентность decide на уровне счётчика: concurrent decide не должны «пробивать» лимит заметно сверх правила (в рамках выбранного алгоритма).

**Nice to have**

- Soft throttle / shadow mode (считать, но не deny)
- fail-closed на critical endpoints
- Local token lease cache для hot keys (снижение Redis QPS)

**Вне скоупа**

- WAF / bot detection / captcha
- Billing quotas / деньги
- UI дашборд
- Per-tenant биллинг-планы как продукт
- Dynamic anomaly ML

## 3. Нефункциональные требования

| NFR | Цель |
|-----|------|
| Latency decide | p99 **&lt; 5–10 ms** (hot path) |
| Throughput | peak **~50k decide/s** |
| Availability | лимитер не должен быть единственной точкой полного outage без явной политики |
| Accuracy | допустима небольшая погрешность (fixed/sliding window trade-off); over-allow хуже для security-sensitive, over-deny — для UX |
| Consistency | не strong global; eventual между репликами store ок, если алгоритм это учитывает |
| Durability | state лимитов **эфемерный** (ок потерять при рестарте Redis) — лимиты не ledger |
| Isolation | hot key (один user/IP) не должен класть весь кластер |

**Политика при сбое store (вводная интервьюера):**
- **fail-open** — при ошибке Redis → **allow** (+ метрика/алерт)
- Рекомендация в дизайне: поверх этого — грубый **local in-process** лимит (safety net), без координации между инстансами
- **fail-closed** только точечно для critical paths (nice-to-have)

## 4. Оценка нагрузки (capacity estimation)

**Decide QPS:** peak **50k req/s** через gateway ≈ 50k decide/s.  
Avg matched rules ≈ **1.5** → Redis Lua ≈ **75k eval/s**.

**Redis memory (order-of-magnitude):**

```text
active_keys ≈ unique (rule, subject) в TTL-окне
assume 5M hot keys × ~128 B ≈ 640 MB
+ replicas / fragmentation → планировать ~2–4 GB на кластер с запасом
```

**Сеть:** Lua round-trip ~200–400 B → при 75k/s ≈ **15–30 MB/s** aggregate (размазано по шардам).

**Gateway pods:** CPU на match rules дешёвый; bottleneck = Redis RTT.  
При RTT 0.5–1 ms p50 — ок для p99 5–10 ms (с запасом на GC/queue).

**MySQL / Kafka (control plane):** правила меняются редко (<< 1 QPS writes); CDC/Kafka нагрузка ничтожна vs decide path.

## 5. High-level архитектура

Схема: [`diagrams/rate-limiter.jpg`](./diagrams/rate-limiter.jpg)

**Компоненты (как на схеме):**

| Компонент | Роль |
|-----------|------|
| Client | получает `X-RateLimit-Remaining` / `Reset` / `Retry-After` |
| nginx | TLS termination, geo restriction, blacklist (edge, **до** RL) |
| gateway / rate-limiter | library: match rules → Redis Lua → allow/deny + headers |
| Redis Cluster (shard-1…n) | token buckets; Lua на primary шарда |
| domain-service1/2 | upstream после allow |
| rule db (MySQL) | SoT правил |
| CDC → Kafka | push `rule change` на каждый gateway |

**Decide path:**

```text
Client → nginx (TLS / geo / blacklist)
      → Gateway rate-limiter
           1. extract subjects: userId?, apiKey?, IP, METHOD:path
           2. RulesSnapshot.match → ordered rules
           3. for each rule (AND): Lua on Redis Cluster primary
           4. deny → 429 + headers; allow → domain-service*
           5. Redis error → local safety + fail-open + alert
```

**Control plane (низ схемы):** rule db → CDC → Kafka → gateway (`rule change` → in-memory snapshot).

## 6. API

### Клиентский контракт (через gateway)

```http
GET /api/v1/orders
X-Api-Key: ...

# allow — обычный ответ upstream + headers:
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1699999999

# deny:
HTTP/1.1 429 Too Many Requests
Retry-After: 3
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1699999999
```

При нескольких правилах: при **deny** — headers от deny-rule (первый по `priority`); при **allow** — от правила с самым узким `remaining/limit` (§8.3). `X-RateLimit-Reset` ≈ `now + retry_after` (семантика TB, не граница fixed window).

### Admin API (control plane)

```http
PUT    /admin/rate-limit/rules/{id}
GET    /admin/rate-limit/rules
DELETE /admin/rate-limit/rules/{id}
```

Тело: `dimension`, `match_pattern`, `limit_count`, `window_seconds` (или `refill_per_sec`), `priority`, `enabled`.  
Пишет в MySQL; доставка на gateway через CDC (не синхронный push из admin).

### Внутренний decide (если library API)

```text
DecideRequest  { subjects: Map<Dimension,String>, method, path }
DecideResponse { allowed, limit, remaining, retryAfterMs, ruleId? }
```

## 7. Модель данных

### Правила (MySQL)

```text
rate_limit_rules
  id
  name
  dimension     -- USER_ID | IP | API_KEY | ...
  match_pattern -- опционально: endpoint/method glob, e.g. "POST:/api/v1/orders*"
  limit_count   -- capacity bucket / max tokens (e.g. 100)
  refill_per_sec -- или window_seconds + limit → derive refill
  priority      -- порядок применения при нескольких правилах
  enabled
  updated_at
  version       -- монотонный; идемпотентность apply на gateway + метрики
```

Индекс: `id`, `version`. Topic Kafka: `rate-limit.rules` (compacted by `rule_id` — опционально для bootstrap без MySQL).

### Counters (Redis Cluster)

Один ключ на (rule, subject) — HASH или STRING+meta. Hash-tag в ключе → один hash slot (см. §8.2).

```text
key:  rl:{{{rule_id}:{subject}}}   -- hash-tag = {rule_id:subject}
fields (HASH):
  tokens     -- float/double as string
  updated_at -- unix ms last refill
TTL: ~ 2–3 × (capacity / refill_rate)  -- ключ умирает, если subject «остыл»
```

## 8. Deep dive

### 8.1 Доставка правил в gateway — push CDC → Kafka (зафиксировано)

**Цель:** decide path не ходит в MySQL; смена правила доезжает за **секунды (обычно &lt;1–2 s)**, не за poll interval.

```text
Admin → INSERT/UPDATE/DELETE rate_limit_rules (MySQL)
         │
         ▼
      CDC (Debezium / Canal) ──► Kafka topic rate-limit.rules
         payload: { op: UPSERT|DELETE, rule_id, version, fields... }
         key: rule_id   (для compaction)

Gateway instance #1  ──► consumer group: rl-rules-{pod_id}   ─┐
Gateway instance #2  ──► consumer group: rl-rules-{pod_id}   ─┼─► каждый читает ВСЕ события
Gateway instance #N  ──► consumer group: rl-rules-{pod_id}   ─┘
         │
         ▼
  apply to in-memory map[rule_id] → atomic RulesSnapshot swap
  last_applied_version / last_event_ts → metrics

Decide path (без I/O к MySQL/Kafka):
  rules = RulesSnapshot.current()
  match → Redis Lua
```

**Почему отдельный consumer group на инстанс:** обычный shared group размазал бы партиции — часть инстансов не увидела бы часть правил. Нужна **broadcast-семантика**: каждый pod — своя копия потока.

**Bootstrap (обязателен):**

1. Startup: **полный snapshot** из MySQL `SELECT * WHERE enabled` (или читать compacted topic с начала / snapshot offset).
2. Запомнить `bootstrap_version` / binlog position / Kafka offset.
3. Подписаться на Kafka **с offset после snapshot** (или с earliest + идемпотентный apply по `version`).
4. Readiness **true** только после snapshot + consumer running.

Без bootstrap пустой RAM до первого CDC-события = дыра в лимитах.

**Apply / идемпотентность:**

- Kafka at-least-once → apply по `rule_id` + `version`: игнор если `event.version <= local.version`.
- DELETE / `enabled=false` → удалить из map.
- Immutable snapshot + atomic swap (как раньше).

**Параметры:**

| Параметр | Значение | Смысл |
|----------|----------|--------|
| Delivery lag target | **p99 &lt; 2–5 s** после commit в MySQL | CDC + Kafka |
| Consumer | 1 per gateway pod, own group | broadcast |
| Topic | compacted optional | упрощает cold start без MySQL |
| Reconcile safety net | редкий full poll **раз в 5–15 min** (или по алерту) | ловит missed CDC / drift |
| In-memory | map + atomic snapshot | decide lock-free |

**Failure modes:**

| Сбой | Поведение |
|------|-----------|
| Kafka / CDC down | last snapshot; алерт `rules_stale` если нет событий **и** reconcile давно / heartbeat CDC lag высок |
| MySQL down | CDC молчит; snapshot живёт; admin не может менять правила |
| Дубликаты событий | игнор по `version` |
| Out-of-order (редко) | только apply если `version` новее |
| Новый pod | bootstrap MySQL → catch-up Kafka; не готов без snapshot |
| Drift (пропуск CDC) | periodic reconcile vs MySQL |

**Trade-off vs poll:**

| | Poll | CDC → Kafka (выбрано) |
|--|------|------------------------|
| Сложность | низкая | выше (CDC, topic, per-pod consumers) |
| Lag | 10–30 s | секунды / субсекунды |
| Нагрузка на MySQL | периодический SELECT всеми pods | только bootstrap + редкий reconcile |
| Ops | проще | нужна зрелая Kafka/CDC платформа |

**Связь с decide:** без изменений — subjects → match snapshot → Redis Lua (AND) → headers.

### 8.2 Token Bucket + Lua + Redis Cluster (зафиксировано)

#### Алгоритм Token Bucket

Параметры правила: `capacity` (max tokens), `refill_per_sec` (скорость восстановления).

На каждый decide:

1. `elapsed = now - updated_at`
2. `tokens = min(capacity, tokens + elapsed * refill_per_sec)`
3. если `tokens >= 1` → `tokens -= 1`, **allow**; иначе **deny**
4. сохранить `tokens`, `updated_at = now`
5. вернуть: `allowed`, `remaining=floor(tokens)`, `retry_after_ms` (сколько ждать до 1 токена)

Burst = `capacity`. Средний rate = `refill_per_sec`.  
Пример: 100 req / 60s → `capacity=100`, `refill=100/60 ≈ 1.67/s`.

Почему TB, не fixed window: нет «двойного burst» на границе окна; плавный refill.

#### Почему Lua (атомарность)

Без скрипта: `HGET` → compute → `HSET` — race между gateway-инстансами → over-allow.  
**Lua на primary шарда** выполняется как один атомарный шаг (single-threaded Redis).

Псевдо-Lua (один KEYS[1]):

```lua
-- KEYS[1]=bucket key; ARGV: capacity, refill_per_sec, now_ms, cost(1)
local data = redis.call('HMGET', KEYS[1], 'tokens', 'updated_at')
local tokens = tonumber(data[1]) or tonumber(ARGV[1])  -- cold start = full
local updated = tonumber(data[2]) or tonumber(ARGV[3])
local cap, refill, now, cost = tonumber(ARGV[1]), tonumber(ARGV[2]), tonumber(ARGV[3]), tonumber(ARGV[4])
tokens = math.min(cap, tokens + ((now - updated) / 1000.0) * refill)
local allowed = 0
local retry_ms = 0
if tokens >= cost then
  tokens = tokens - cost
  allowed = 1
else
  retry_ms = math.ceil((cost - tokens) / refill * 1000)
end
redis.call('HSET', KEYS[1], 'tokens', tokens, 'updated_at', now)
redis.call('PEXPIRE', KEYS[1], ARGV[5])  -- TTL ms
return {allowed, math.floor(tokens), retry_ms}
```

Gateway мапит return → 429 / headers (`X-RateLimit-Remaining`, `Retry-After`).

#### Redis Cluster: hash ring + слоты

Cluster делит keyspace на **16384 hash slots**; слоты распределены по нодам (consistent hashing / slot map, клиент знает mapping).

```text
CRC16(key) mod 16384 → slot → node (primary)
```

Масштаб write QPS: разные subjects → разные слоты → разные primary → горизонтальный шардинг.  
Decide всегда на **primary** слота (replica — read-only / failover).

#### Hash-tag — обязателен для Lua в Cluster

Lua в Cluster может трогать ключи **только одного slot**.  
Синтаксис: часть ключа в `{...}` участвует в хеше слота:

```text
rl:{user:42}:rule:7     → slot = hash("user:42")
rl:{user:42}:meta       → тот же slot
```

**Наш контракт v1:** один bucket = **один ключ** на decide → Lua с `KEYS[1]` всегда safe.  
Hash-tag всё равно ставим как `{rule_id:subject}`, чтобы:

- явно зафиксировать sharding key = subject (+ rule);
- оставить место для связанных ключей (cooldown, shadow) в том же слоте без CrossSlot.

```text
ключ: rl:{{{ruleId}:{dimension}:{subjectId}}}
пример: rl:{r7:user:u42}
```

Без hash-tag / с несколькими ключами в разных слотах → **CROSSSLOT** ошибка.

#### TTL

Ключ эфемерный: `PEXPIRE ≈ 2–3 × (capacity / refill)` (время полного восстановления + запас).  
Неактивный subject исчезает → экономия памяти. Cold start в Lua = полный bucket (лёгкий over-allow после expiry — ок для rate limit).

#### Hot key

Один вирусный `subject` бьёт **один** primary слота → CPU/сеть этой ноды. Cluster ring **не** спасает от hot key (все запросы одного ключа на одном слоте).

Митигации (говорить на интервью, не обязательно v1):

- local token cache / lease на gateway (short TTL) — снижает Redis QPS, повышает over-allow;
- dual-write split buckets + агрегация (сложнее, погрешность);
- алерт на per-key QPS / latency шарда.

Изоляция: hot user не кладёт *весь* кластер, только свой слот/ноду — уже лучше single Redis.

#### Fail-open на этом слое

Redis timeout / CLUSTERDOWN / OOM на decide → **allow** + метрика `rl_store_errors` (вводная интервьюера). Опционально local in-process cap.

#### Capacity (салфетка)

См. §4: ~75k Lua/s при 50k decide; память порядка GB при миллионах hot keys.

### 8.3 Multi-dimension AND + headers (зафиксировано)

1. Из request: `IP`, `USER_ID` (если auth), `API_KEY` (если есть), `METHOD:path`.
2. Snapshot: все enabled rules, у которых dimension subject присутствует **и** `match_pattern` совпал (или pattern пустой = all).
3. Сортировка по `priority` asc.
4. Для каждого rule — отдельный Redis key / Lua. **AND:** первый deny → стоп, 429; остальные buckets **не** списываем после deny (чтобы deny IP не жег user-budget зря).  
   Альтернатива «списать все» — отвергли для v1.
5. Headers: от deny-rule; при полном allow — от rule с **min(remaining/limit)** (самое узкое место) или от последнего evaluated — зафиксировали: **самое узкое remaining**.

### 8.4 Local safety net при fail-open (зафиксировано)

Когда Redis недоступен:

```text
global fail-open allow
  + per-instance in-process TokenBucket(key=subject, cap=k, refill=r)
  если local deny → всё же 429 (защита своего pod)
  если local allow → пропустить + metric rl_fail_open_total
```

Local лимит **слабее** глобального (например 2–5× мягче или доля `global/N_pods` грубо) — координации нет, over-allow между pods остаётся; цель — не «дырка бесконечность» на одном инстансе.

### 8.5 Observability (must)

| Метрика / сигнал | Зачем |
|------------------|--------|
| `rl_decide_latency` | p99 vs NFR |
| `rl_allow` / `rl_deny` by rule | продукт / abuse |
| `rl_store_errors` / fail-open | алерт немедленный |
| `rl_rules_stale` / CDC lag | control plane |
| Redis shard CPU / hot key | capacity |

## 9. Масштабирование

| Рост | Что ломается первым | Что делать |
|------|---------------------|------------|
| ×10 (500k decide/s) | Redis CPU / сеть на шардах | больше primary nodes (перераздача slots); pipeline/connection pool; ужать matched rules |
| Hot key ×N | один primary | local lease cache; опционально split key (погрешность); isolate heavy tenants |
| Rules ×100k | match CPU / snapshot size | индекс по path prefix; шардирование rules по сервису |
| Gateway pods ×N | Kafka consumer groups ×N (broadcast) | ок до сотен pods; дальше — fan-out сервис / shared poll + push |
| Redis failover | краткий spike fail-open | replica promotion; клиент retry с jitter; local safety |

Вертикальный потолок одной Redis-ноды (~100k+ простых ops) → Cluster с запасом шардов под 75k Lua уже на старте (например 3–6 primary).

## 10. Надёжность и failure modes

| Сбой | Влияние | Митигация |
|------|---------|-----------|
| Redis timeout / down | нет глобального лимита | fail-open + local safety + alert |
| Redis primary failover | краткий error spike | Cluster redirect; retry 1×; fail-open |
| Hot key | деградация одного шарда | мониторинг; local cache; throttle tenant |
| MySQL down | нельзя менять правила | snapshot живёт; CDC idle |
| CDC / Kafka down | правила stale | last snapshot; `rules_stale`; reconcile poll |
| Gateway OOM / restart | краткая дыра до bootstrap | readiness gate; быстрый snapshot |
| Clock skew gateway vs Redis | TB использует `now` из gateway ARGV | NTP; допустима малая погрешность refill |
| Lua CROSSSLOT | decide fail | один ключ + hash-tag контракт |
| Wrong rule deploy | массовый deny/allow | versioned rules; быстрый rollback в MySQL + CDC |

## 11. Trade-offs и альтернативы

| Решение | Выбрано | Альтернатива | Почему |
|---------|---------|--------------|--------|
| Размещение | library в gateway | центральный RL service / sidecar | −1 hop latency; проще headers; минус — blast radius кода |
| State | Redis Cluster | MySQL counters / local only | latency + shared state; MySQL не тянет 50k |
| Алгоритм | Token Bucket + Lua | Fixed/Sliding window, GCRA | burst + smooth refill; Lua = atomic |
| Правила delivery | CDC → Kafka broadcast | poll MySQL | низкий lag; платформа Kafka уже ок |
| Redis down | fail-open + local | fail-closed | вводная интервьюера: API > краткий over-allow |
| Multi-rule | AND, stop on first deny | OR / score | типичный gateway: IP и user оба должны пройти |
| Exactness | almost-exact | strong global lock | p99 важнее; Redlock избыточен |

## 12. Follow-ups (после review)

См. [`REVIEW.md`](./REVIEW.md). Кратко:

1. **Poll vs CDC:** на собесе вести с poll; CDC — когда нужен низкий lag / меньше MySQL fan-out.
2. **IP rotate abuse:** держится user/apiKey rule; IP-only недостаточен → multi-dimension AND.
3. **Reshard:** MOVED/ASK + retry; ключ целиком; краткий fail-open ок.
4. **Billing-critical:** точечный fail-closed, не глобально.
5. **500k × 3 rules:** меньше matched rules, больше Redis primaries, local lease на hot keys.
