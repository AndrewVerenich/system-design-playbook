# ADR 0002 — Генерация short code: Snowflake → Base62

- **Status:** Accepted
- **Date:** 2026-08-20
- **Round:** [00-url-shortener](../BRIEF.md)
- **Deciders:** system-design-playbook (sample)

## Context

Нужно генерировать короткие URL-safe коды (`code`) для mapping `code → long_url`.

Ограничения:

- один long URL может иметь **несколько** short links (кампании);
- write ~40/s avg, peak ~400/s — не экстремально, но нужна простая уникальность;
- коды должны быть короткими (~7–8 символов) и не тривиально предсказуемыми для перебора «секретных» ссылок (частично закрываем rate limit + размером пространства);
- custom alias — отдельный путь, но делит тот же unique space `code`.

## Decision

1. Берём распределённый **Snowflake ID** (datacenter + worker + sequence).  
2. Кодируем в **Base62** (`a-zA-Z0-9`) → short `code` фиксированной длины (8).  
3. Пишем в БД с `PRIMARY KEY (code)` / unique constraint.  
4. Custom alias **не** генерируем так: валидируем и вставляем как есть; конфликт → `409`.

```text
snowflake() → 64-bit id → Base62 → code → INSERT url_mapping
```

## Consequences

**Plus**

- практически без коллизий при insert (в отличие от truncate-hash);
- можно иметь много short на один long URL;
- высокая скорость генерации без центрального `SELECT max(id)`;
- хорошо стыкуется с шардированием позже (`hash(code)`).

**Minus**

- нужна аккуратная выдача worker/datacenter id (иначе дубли ID);
- последовательные ID слегка предсказуемы → не полагаемся на obscurity для секретов; для confidential links — длиннее code или auth (follow-up);
- зависимость от clock/ID service operationally (митигируем: ID gen в процессе, не один SPOF).

## Alternatives considered

| Alternative | Why not (now) |
|-------------|----------------|
| **Hash(longUrl) + truncate** | коллизии; один short на long URL — противоречит FR |
| **Random 7–8 chars + retry on conflict** | работает, но retries и hotspot на unique index при росте; хуже объяснять гарантии |
| **Центральный SQL SEQUENCE / ticket server** | проще на одном DC, но bottleneck и сложнее multi-instance |
| **Только UUID** | слишком длинный для UX short URL |

## References

- [DESIGN.sample.md §8.1](../DESIGN.sample.md)
- В platform-репо: паттерн `unique-id-generator`
