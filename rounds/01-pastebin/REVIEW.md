# Раунд 01 — Review (финальный)

> Reviewer: interviewer (AI)  
> Дата: 2026-08-21  
> Документ: [`DESIGN.md`](./DESIGN.md) + [`diagrams/high-level.png`](./diagrams/high-level.png)

---

## Вердикт

**Hire / Strong для первого самостоятельного раунда.**

Каркас интервью собран end-to-end: FR → NFR → capacity → архитектура → API/модель → deep dive → scale/reliability → trade-offs.  
Решения в целом соответствуют нагрузке (нет лишней Kafka/saga). Сильная сторона сессии — умение поправлять дизайн по follow-up (RYOW через Redis, отказ от CDN, S3 на ×100, reuse id после expiry).

На живом 45-мин собесе этого уровня хватило бы на solid pass.

---

## Strengths

1. **Скоуп дисциплинированный** — public-by-link, без users/search; must чёткий.
2. **NFR с цифрами** и поправка CAP → read-your-writes (не наивный «просто AP»).
3. **Capacity** с формулами; отдельно avg vs peak для storage — после коррекции выглядит senior-like.
4. **MySQL + MEDIUMTEXT** обоснован для max 1 MB / avg 5 KB.
5. **Архитектура читаемая:** Nginx → Gateway(RL) → Paste Service → Redis + master/replica + cleanup.
6. **RYOW закрыт** через SET on write — ключевой must-fix раунда.
7. **Expiry:** soft-check на read + hard-delete cron — правильный паттерн.
8. **ID:** random 12 + retry + осознание **reuse после hard-delete** vs Snowflake — зрелый trade-off.
9. **Scale:** ×10 эволюция vs ×100 S3 для blob — сильный ответ.
10. **Отказ от CDN для paste** — осознанно (TTL/stale); схема без CDN согласована с решением.
11. Диаграмма и документ в целом **согласованы** (после правок).

---

## Gaps / risks (не блокеры, но добить)

1. **Схема vs текст ID:** на PNG всё ещё фигурируют snowflake *и* random; в `DESIGN.md` выбран **только random 12 + retry**. Подровнять подпись на диаграмме.
2. **`ttl_seconds` + `expires_at`:** в доке оба поля; на схеме только `TTL`. Достаточно `expires_at` (+ опционально хранить исходный ttl для UI).
3. **Нет явного rate-limit ключа** — по IP? по /pastes? На интервью спросят одну фразу.
4. **Observability** почти нет — SLI: create/get latency, Redis hit ratio, replica lag, cleanup lag, 404 rate.
5. **SSRF / abuse:** paste публичный; лимит размера есть, но не сказано про content-type/validate UTF-8 / blocklist (мелочь для v1).
6. **Graceful: Redis SET fail** описан в §8.3 — хорошо; на схеме этого edge нет (нормально).
7. Статус в шапке ещё `draft` → после этого review можно ставить **`reviewed` / `accepted`**.

---

## Follow-up questions (как добивали бы на собесе)

1. Viral paste: 50k rps на один `id`. Redis один ключ/slot горит. Что делаешь за час и за неделю?  
2. Burn-after-read: два параллельных GET — кто побеждает, как сделать exactly-once «сжечь»?  
3. Cleanup удалил строку, Redis ещё держит значение до своего TTL — что увидит клиент и как синхронизировать?  
4. Нужен GDPR «удалить paste сейчас» — invalidate Redis + delete master; replica lag — как не отдать удалённое?  
5. Почему Application Gateway, а не rate limit в Nginx/Paste Service?

### Эталонные тезисы ответов

1. Local cache / replicate hot key / more Redis CPU; не DB. Долгосрочно — dedicated hot-key path, optionally edge только если очень надо (мы CDN отвергли — тогда vertical Redis + app cache).  
2. `DELETE`/`UPDATE burned` в master в транзакции + `GET` с lock или atomic Redis GETDEL + async DB; один 200, второй 404.  
3. На read всегда сверять `expires_at`; при expiry — DEL Redis; Redis TTL ≤ expires. Истёкший в Redis при проверке → 404 + invalidate.  
4. Delete master + DEL Redis; на read miss если replica ещё отдаёт — проверять tombstone/version или читать master для «negative cache».  
5. Gateway — единая точка RL для многих сервисов later; в v1 можно было в Nginx — сказать «вынес явно для ясности».

---

## Оценка по рубрике интервью

| Критерий | Оценка |
|----------|--------|
| Clarifying / scope | Strong |
| NFR + capacity | Strong |
| High-level design | Strong |
| Deep dive (ID, cache, TTL) | Strong |
| Scaling | Strong |
| Reliability | Good → Strong (после правок по Redis/replica) |
| Communication / trade-offs | Strong |
| Overengineering | Хорошо сдержан |

**Итог:** Strong Hire на уровень senior для задачи Pastebin.
