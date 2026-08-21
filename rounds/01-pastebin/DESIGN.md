# Система: Pastebin

> Раунд: 01  
> Статус: **reviewed** (финальный review в [`REVIEW.md`](./REVIEW.md))  

## 1. Уточняющие вопросы

Что спросил у интервьюера / какие assumptions зафиксировал:

- Paste **публичный по ссылке**: кто знает `id`, тот читает. Без auth / паролей в v1.
- Пользователи, поиск, редактирование — вне скоупа этого раунда.
- Burn-after-read — nice-to-have, в must не входит (можно вернуться later).

## 2. Функциональные требования

**Must have (v1)**

1. **Создать paste:** принять текстовое тело, вернуть короткий публичный URL / `id`.
2. **Прочитать paste:** по `id` вернуть текст (и при необходимости метаданные: createdAt, expiresAt, language).
3. **TTL / expiry:** при создании задаётся срок жизни; после истечения paste недоступен (read → not found).
4. **Лимит размера:** один paste не больше **1 MB**.
5. **Публичный доступ по ссылке:** секретность = знание `id` (не ACL).

**Nice to have (не проектируем глубоко сейчас)**

- Burn-after-read (удалить после первого успешного чтения)
- Поле `language` для подсветки на клиенте

**Вне скоупа**

- Регистрация / логин / «мои pastes»
- Текстовый поиск
- Редактирование и версии
- Приватные paste / пароли / ACL
- Комментарии, лайки, соцфункции
- Загрузка бинарных файлов (только текст)
- Биллинг, команды, enterprise

## 3. Нефункциональные требования

| NFR | Цель |
|-----|------|
| DAU (ориентир) | ~1M |
| Throughput create | avg ~20/s, peak **50/s** |
| Throughput read | avg ~200/s, peak **500/s** (≈10:1) |
| Max size paste | **1 MB** |
| TTL | default **30 дней**; опции 1h / 1d / 7d / 30d; «навсегда» в v1 нет |
| Availability read | **99.9%** |
| Availability create | **99.9%** |
| Latency read | p99 **&lt; 100 ms** |
| Latency create | p99 **&lt; 300 ms** |
| Durability | после успешного `201` mapping не теряется |
| Consistency | **read-your-writes** после create; между репликами async ok для остальных читателей |
| CAP (практически) | на write лучше отказать, чем отдать ссылку на несуществующие данные; на read — высокая availability (cache / replicas) |

## 4. Оценка нагрузки (capacity estimation)

**QPS (из NFR):**
- create: avg ~20/s, peak ~50/s
- read: peak ~500/s

**Storage (steady-state при TTL 30 дней):**
```text
avg creates: 20/s × 86_400 × 30 × 5 KB ≈ 259 GB «живых» тел
+ метаданные/индексы ≈ ×1.3–1.5 → ориентир ~350–400 GB
peak-непрерывный месяц (50/s) ≈ 648 GB — upper bound, не baseline
```

**Bandwidth (read peak):**
```text
500 rps × 5 KB ≈ 2.5 MB/s ≈ 20 Mbit/s
```

**Cache:** Redis — cache-aside на read + **SET on write** (чтобы закрыть read-your-writes). Кэширование paste-ответов на CDN **не используем**.

## 5. High-level архитектура

Схема: [`diagrams/high-level.png`](./diagrams/high-level.png)

```text
Client
  → Nginx (TLS, static JS/CSS/HTML)
  → Application Gateway (rate limiting)
  → Paste Service
       ├─ ID generator (random 12 + retry)
       ├─ Redis (cache)
       ├─ MySQL master (writes + cleanup deletes)
       └─ MySQL replica (read on cache miss)
  Cleanup cron (hourly) → delete expired on master
```

**Write path:**
1. `POST /pastes` → Nginx → Gateway (rate limit) → Paste Service  
2. Сгенерировать `id` (random 12, retry при unique conflict)  
3. `INSERT` в **MySQL master**  
4. `SET` в **Redis** (write-through / set-on-write)  
5. `201` + URL `host/{id}`  
6. Репликация master → replica (async)

**Read path:**
1. `GET /pastes/{id}` → Nginx → Gateway → Paste Service  
2. Redis GET (cache-aside)  
3. Miss → read **replica** → проверка expiry (soft)  
4. Если на replica нет / подозрение на lag → **fallback master**  
5. Ответ клиенту / 404 если истёк или не найден  

**Read-your-writes:** сразу после create значение в Redis; при сбое SET — fallback read с master.

**Почему без CDN для paste:** динамический контент + TTL/expiry; риск отдать stale/удалённый paste; при нашей нагрузке достаточно Redis + origin. Nginx отдаёт только **статику** (JS/CSS/HTML), не тело paste.

## 6. API

```http
POST /pastes
Content-Type: application/json

{
  "text": "...",          // max 1 MB
  "ttlSeconds": 2592000   // optional; default = 30 days
}

→ 201 Created
{
  "id": "a1b2c3d4e5f6",
  "url": "https://paste.example/a1b2c3d4e5f6",
  "expiresAt": "2026-09-20T12:00:00Z"
}

→ 413 Payload Too Large
→ 429 Too Many Requests
```

```http
GET /pastes/{id}
→ 200 OK
{
  "id": "a1b2c3d4e5f6",
  "text": "...",
  "createdAt": "...",
  "expiresAt": "..."
}

→ 404 Not Found   // нет записи или истек TTL
```

Публичный URL может совпадать с `GET /{id}` на nginx → тот же handler.

## 7. Модель данных

**MySQL `pastes`:**

| Column | Type | Notes |
|--------|------|--------|
| `id` | `VARCHAR(12)` PK | random Base62/alphanumeric |
| `text` | `MEDIUMTEXT` | тело, max 1 MB на уровне API |
| `created_at` | `DATETIME(3)` | |
| `ttl_seconds` | `BIGINT` | TTL в секундах с момента create |
| `expires_at` | `DATETIME(3)` | вычисляется при insert: `created_at + ttl`; удобнее для cleanup/index |

Индексы:
- PK(`id`)
- INDEX(`expires_at`) — для cleanup: `DELETE WHERE expires_at < now() LIMIT …`

**Redis:**
```text
key:   paste:{id}
value: { text, createdAt, expiresAt }
TTL:   min(time_until_expires, policy_cap)  // ключ сам протухает
```

**Cleanup:** cron ~раз в час на master: hard-delete по `expires_at`.  
Корректность чтения не ждёт cron: на read всегда проверяем `expires_at` (и Redis TTL).

## 8. Deep dive

### 8.1 Генерация `id`: random 12 + retry (выбрано)

**Решение:** при create генерируем случайную строку фиксированной длины **12** (алфавит ~Base62), `INSERT`, при duplicate key — retry (несколько попыток).

Почему для v1:
- просто реализовать и объяснить;
- при 50 create/s и пространстве `62^12` коллизии при insert крайне редки;
- не нужен отдельный stateful ID-сервис.

**Алгоритм:**
```text
repeat up to N times:
  id = random(12)
  try INSERT
  on success → break
  on duplicate → retry
```

### 8.2 Коллизии «во времени»: протухший paste и новый с тем же id

Важное наблюдение: при **hard-delete** истекшего paste id снова свободен.  
Random теоретически может выдать **тот же id** → старая закладка откроет **другой** текст (семантика URL сломалась).

| Подход | Повторное использование id | Сложность |
|--------|----------------------------|-----------|
| **Random + hard delete** | возможно (редко при большой длине) | проще |
| **Snowflake / monotonic id** | практически нет | чуть сложнее ops |
| Random + **не освобождать id** (tombstone / soft-delete) | нет | средний вариант |

**Наш выбор в v1:** random 12 + retry (простота).  
Риск reuse снижаем:
1. Большое пространство id (12 символов).  
2. Cleanup может **soft-delete** / держать tombstone `id` + `expired` без выдачи текста (id занят навсегда) — если продукт запрещает «переезд» URL.  
3. Если требование «id никогда не указывает на другой контент» станет жёстким → перейти на **Snowflake** (или ULID): id не переиспользуется после expiry.

Snowflake сам по себе не «меньше коллизий на insert», а **не переиспользует** значения → нет сюрприза «старая ссылка = новый paste». Это правильный аргумент *за* snowflake, даже если в v1 берём random.

### 8.3 Redis SET on write и сбой Redis

Если master commit успешен, а Redis SET нет: GET может уйти в replica и поймать короткий lag.  
Митигация: retry SET; при miss на «свежем» create — fallback read с **master**; create всё равно возвращаем 201 (источник истины — MySQL).

## 9. Масштабирование

Baseline: peak 50 create/s, 500 read/s, ~350–400 GB, 1× MySQL master + replica, Redis, Paste Service.

### ×10 (≈500 create/s, ≈5k read/s, ~3–4 TB)

Что упирается и в каком порядке действовать:

| Порядок | Узкое место | Действие |
|---------|-------------|----------|
| 1 | **Read path** — не столько «реплика сразу», сколько **Paste Service + Redis** при 5k rps; реплика горит при **низком hit ratio** / cold traffic | HPA Paste Service; Redis scale (cluster); поднять hit ratio; local in-process cache для ultra-hot keys; больше read replicas если miss’ы частые |
| 2 | **Write path** — master IOPS / binlog / fsync при 500 insert/s с телами до 1 MB | Пока часто ещё тянет один master; тюнинг batch/innodb; вертикаль; не шард первой реакцией |
| 3 | **Storage** — диск и backup window | Больше volume; TTL cleanup агрессивнее; мониторинг free space |
| 4 | Cleanup job | Batched `DELETE … LIMIT`, индекс по `expires_at` |

Важно: при нормальном Redis большинство GET **не** ходят в реплику. Реплика — bottleneck на miss/cold, не на каждый read.

### ×100 (≈5k create/s, ≈50k read/s, десятки TB)

Уже не «добавить реплик»:

1. **Тело paste → object storage (S3)** + в MySQL только метаданные (`id`, `s3_key`, `expires_at`, size). Проще, чем шардировать жирные `MEDIUMTEXT` на много MySQL.
2. MySQL (meta) при необходимости **шардировать по `hash(id)`** или оставить один meta-store дольше, чем blob-store.
3. Redis cluster; при необходимости local cache на инстансе для ultra-hot keys.
4. ID generation и rate limit — отдельные мощные пути.

**Решение на горизонте роста:** согласились — **S3 для текста предпочтительнее раннего шардирования MySQL с blob внутри**. В v1 оставляем MySQL `MEDIUMTEXT` (avg 5 KB, max 1 MB, проще).

## 10. Надёжность и failure modes

| Сбой | Влияние | Митигация |
|------|---------|-----------|
| **Redis down** | Create в MySQL ок, но **нет SET** → RYOW может сломаться (GET → replica → lag/404). Read медленнее (всегда DB) | Create всё равно 201 (source of truth = MySQL). Retry SET. На cache miss / 404 с replica — **fallback read master**. Алерт на Redis |
| **Replica lag 30s**, Redis miss | Клиент может получить 404 или устаревшее для «недавно» записанного; для старого paste — обычно ок, если replica догнала | **Не лечится одним ↑ TTL Redis** (TTL снижает miss, но не lag на cold miss). Fallback: miss/404 → master; следить `Seconds_Behind_Source`; для RYOW опираться на Redis SET on write |
| **Master down ~2 min** | **Create недоступен**; **read доступен** из Redis + replica | Failover/promote replica; status page; очередь create не обязательна в v1 |
| ID collision | Retry create | N retries → 503 |
| Cleanup отстаёт | Диск растёт; read всё ещё корректен если soft-check `expires_at` | Индекс + batched delete; алерт на lag cleanup |

### Зафиксированные ответы раунда

1. Redis лежит → create без прогрева кэша → риск неконсистентного read-after-write; чиним fallback на master + retry SET.  
2. Lag replica + miss → ↑ Redis TTL **помогает косвенно** (меньше miss), но правильный ответ — **master fallback / уменьшить lag**.  
3. Master down → write нет, read да (cache + replica).

## 11. Trade-offs и альтернативы

| Решение | Выбрано | Альтернатива | Почему |
|---------|---------|--------------|--------|
| Store | MySQL (`MEDIUMTEXT`) | S3 + meta в MySQL | max 1 MB, avg 5 KB — БД проще |
| Cache | Redis set-on-write + cache-aside | только replica | нужен read-your-writes |
| Edge cache paste | **нет** (не кэшируем paste на CDN) | CDN для popular GET | TTL/expiry + риск stale; хватает Redis |
| ID | **random 12 + retry** | Snowflake | проще в v1; snowflake лучше против reuse id после expiry |
| Expiry | soft check on read + hourly hard delete | только TTL в Redis | Redis ≠ source of truth |
| Gateway | rate limit | только nginx | явный слой лимитов |

## 12. Follow-ups (после review)

См. полный список и эталонные тезисы в [`REVIEW.md`](./REVIEW.md).

Кратко зафиксированные темы добивки:
1. Viral hot key без CDN — Redis / local cache.  
2. Burn-after-read и гонка двух GET.  
3. Redis ещё держит истёкший paste — сверка `expires_at` + DEL.  
4. Немедленное удаление + replica lag.  
5. Зачем отдельный Gateway для rate limit.
