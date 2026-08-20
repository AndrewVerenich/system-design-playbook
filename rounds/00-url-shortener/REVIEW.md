# Раунд 00 — Review (образец)

> Это **эталон review** к [`DESIGN.sample.md`](./DESIGN.sample.md): как выглядит разбор после сдачи дизайна.  
> На следующих раундах сюда попадёт review **твоего** `DESIGN.md`.

---

## Strengths

- Зафиксированы assumptions до архитектуры — не прыжок сразу в Kafka/Cassandra.
- Есть back-of-the-envelope: write/read QPS, storage, оценка cache — цифры связаны с решениями.
- Чёткий split Write vs Redirect под разные SLO.
- Access pattern назван прямо: KV по `code` → отсюда cache-aside + PK lookup.
- Генерация кода обоснована (Snowflake → Base62) против hash/random с аргументами.
- Hot path защищён: Redis, singleflight, local cache для viral; analytics вынесены в async.
- Failure modes таблицей: что деградирует первым (create vs redirect).
- Trade-offs явные (302 vs 301, SQL vs Cassandra first) — не «возьмём всё».
- Follow-up’ы в конце документа показывают готовность к давлению интервьюера.

---

## Gaps / risks

Даже сильный дизайн можно добить на интервью. Типичные «дыры» относительно sample / что ещё спросят:

1. **Multi-region** описан как follow-up, но нет конкретной модели (active-passive vs edge cache only). На FAANG часто углубляют.
2. **SSRF / open redirect abuse** при валидации `longUrl` упомянут кратко — мало деталей (allowlist схем, block private IP).
3. **Идемпотентность create** есть в API, но нет схемы хранения ключей (TTL таблицы / Redis).
4. **Миграция шардов** — consistent hashing назван, план dual-write / миграции живых данных не расписан.
5. **Наблюдаемость**: нет конкретных SLI/алертов (redirect p99, cache hit ratio, create 409 rate).
6. Объём sample больше, чем успеют устно за 45 мин — на живом интервью нужно уметь **сжать** до каркаса, а deep dive отдавать по вопросам.

---

## Follow-up questions

Как интервьюер после «готово, вот архитектура»:

1. Viral link: 2M redirects/min на один `code`. Redis в одном shard/slot горит. Что делаешь за 10 минут и за неделю?
2. Пользователь создал ссылку и сразу открыл — иногда 404. Где race и как гарантировать read-your-writes?
3. Нужно запретить сокращать ссылки на phishing-домены с задержкой &lt; 100 ms на create. Куда вставишь проверку, не убив write SLO?
4. Переезд с одного Postgres на 8 шардов без даунтайма redirect. Опиши шаги.
5. Бюджет: команда из 2 человек, 3 месяца. Что выкинешь из sample-дизайна, а что оставишь обязательно?

---

## Answers / revisions (эталонные тезисы)

1. **Viral:** сразу — local in-process cache / edge; не ходить в DB; scale Redis replica reads для этого key; за неделю — отдельный hot-key path, CDN если допустим cacheable redirect, possibly dedicate cache node for top-N.  
2. **404 after create:** warm Redis на create до `201`, или redirect на short TTL читает primary, или sync replicate mapping в cache.  
3. **Phishing:** async post-create scan + disable flag (не блокировать create синхронно на тяжёлом API); либо sync allowlist/blocklist в Redis (&lt;1 ms). Тяжёлый ML — только async.  
4. **Reshard:** dual-write new shard map, backfill by hash ranges, read fallback old→new, cutover, remove old. Redirect всегда `hash(code)`.  
5. **Budget cut:** один сервис (модули), managed PG + Redis, без Kafka до analytics, без multi-region; оставить: unique code gen, cache-aside, rate limit, 302, unique constraint на alias.

---

## Вердикт (как на интервью)

**Strong hire / Excellent** по содержанию sample.  
Для тренировки: цель — устно за 45–60 мин закрыть 70–80% этой глубины и уверенно ответить на follow-up’ы выше.
