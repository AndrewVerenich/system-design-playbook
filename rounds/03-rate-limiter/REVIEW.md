# Раунд 03 — Review (финальный)

> Reviewer: interviewer (AI)  
> Дата: 2026-10-14  
> Документ: [`DESIGN.md`](./DESIGN.md) + [`diagrams/rate-limiter.jpg`](./diagrams/rate-limiter.jpg)

---

## Вердикт

**Strong Hire** для senior system design (Rate Limiter).

End-to-end каркас закрыт: FR/NFR → capacity → архитектура + схема → API/модель → deep dive (rules push, TB+Lua+Cluster, AND, fail-open) → scale / failures / trade-offs.  
Горячий путь дисциплинированный (RAM + Redis only). Осознанная работа с вводной fail-open + local safety. Схема и текст согласованы.

Минус по «классическому» RL-интервью: **CDC→Kafka для правил** тяжелее, чем нужен большинству (poll 15s часто хватает). Ты это явно trade-off’нул — на собесе важно сразу сказать «начал бы с poll, push — когда lag/ops требуют».

На 45–60 мин живого интервью этого уровня — уверенный pass / strong.

---

## Strengths

1. **Scope** — must/nice/out чёткие; WAF/billing отрезаны.
2. **Fail-open как вводная** принята и усилена local in-process bucket — не «просто allow forever».
3. **Library в gateway** — −1 hop, headers на месте; trade-off vs central service назван.
4. **Token Bucket + Lua** — правильный ответ на atomicity без Redlock.
5. **Redis Cluster + hash-tag / CROSSSLOT** — уровень, который отличает mid от senior.
6. **Hot key честно:** ring не спасает один subject; изоляция на шард + митигации.
7. **Rules delivery:** bootstrap + per-pod consumer group (broadcast) + version idempotency + reconcile — полный control-plane, не «Kafka и надеемся».
8. **AND + stop on first deny** — не жжёт соседние budgets; headers зафиксированы.
9. **Capacity** с QPS → Lua/s → memory/network — салфетка ок.
10. **Диаграмма** читаемая: edge nginx → RL → Redis Lua; снизу CDC pipeline; headers у клиента.
11. **Observability** в must — не забыли после архитектуры.

---

## Gaps / risks (не блокеры)

1. **Overengineering control plane:** BRIEF предупреждал не тащить Kafka/CDC без нужды. На интервью: start with poll → evolve to CDC. В доке trade-off есть — в устной подаче вести с poll.
2. **§6 headers vs §8.3:** в API фраза про «первый deny по priority» чуть путается с «на allow — самое узкое remaining». Смысл в §8.3 верный; подровнять одну строку в §6.
3. **`X-RateLimit-Reset`:** в TB естественнее `Retry-After` / время до 1 токена; unix reset «окна» — наследие fixed window. Уточнить семантику Reset = `now + retry_after`.
4. **Per-pod Kafka CG × N:** при тысячах gateway — cost. В §9 есть fan-out later — ок; держать в голове.
5. **Admin auth / audit** правил не описан (мелочь; admin вне hot path).
6. **EVALSHA + SCRIPT LOAD** на новый Redis node после failover — ops detail, на собесе плюс одной фразой.
7. Статус в шапке `DESIGN.md` был `draft` → ставим **reviewed**.

---

## Follow-up questions (как добивали бы на собесе)

1. Зачем Kafka для правил, если правила меняются раз в час? Когда останешься на poll?
2. Два правила (IP и user), IP deny — user bucket не трогаем. Клиент ретраит с другого IP — ок ли это для abuse?
3. Redis Cluster reshard mid-flight: что с in-flight Lua?
4. Нужен exact global limit на billing-critical API — что меняешь (fail-closed? другой store)?
5. 500k decide/s, avg 3 rules matched — где потолок и что режешь первым?

### Эталонные тезисы ответов

1. Poll проще ops; CDC когда нужен sub-second / меньше load на MySQL от N pods. Начать с poll — нормальный senior ответ.
2. AND без списания после deny — UX/fairness; abuse с IP rotate ловит user/apiKey rule. Чистый IP-only слаб — поэтому multi-dimension.
3. Клиент получает MOVED/ASK; retry; краткий fail-open ок. Lua на одном key — reshard ключа целиком.
4. Точечный fail-closed + жёсткий alert; возможно отдельный path / более строгий local. Не менять весь кластер на fail-closed.
5. Режешь matched rules (1 primary rule), больше Redis primaries, local lease на hot; не MySQL.

---

## Оценка по рубрике интервью

| Критерий | Оценка |
|----------|--------|
| Clarifying / scope | Strong |
| NFR + capacity | Strong |
| High-level design | Strong |
| Deep dive (TB, Lua, Cluster, rules) | Strong |
| Scaling | Strong |
| Reliability / fail-open | Strong |
| Communication / trade-offs | Strong |
| Overengineering | Good (CDC тяжеловат; осознан) |

**Итог:** Strong Hire на уровень senior для задачи Rate Limiter.
