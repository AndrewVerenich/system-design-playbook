# ADR 0001 — Redirect через HTTP 302, а не 301

- **Status:** Accepted
- **Date:** 2026-08-20
- **Round:** [00-url-shortener](../BRIEF.md)
- **Deciders:** system-design-playbook (sample)

## Context

Сервис должен отдавать короткий URL и при переходе делать redirect на оригинальный long URL.  
Нужно выбрать статус ответа: `301 Moved Permanently` или `302 Found`.

Требования, которые влияют на выбор:

- желательна базовая аналитика кликов;
- destination иногда могут менять (ошибка, смена лендинга, disable phishing);
- redirect — hot path, но контроль и наблюдаемость важнее максимального offload на browser/CDN cache.

## Decision

Используем **`302 Found`** для `GET /{code}` в v1.

Заголовки roughly:

```http
HTTP/1.1 302 Found
Location: https://example.com/original
Cache-Control: no-cache
```

## Consequences

**Plus**

- браузеры и CDN не «залипают» навечно на старый Location;
- каждый (или почти каждый) переход может учитываться как click event;
- проще исправить mapping / отключить ссылку.

**Minus**

- больше трафика на origin / Redirect Service, чем при агрессивном 301 + CDN;
- при viral load нужно сильнее опираться на Redis / local cache, а не на browser cache.

## Alternatives considered

| Alternative | Why not (now) |
|-------------|----------------|
| **301** | лучше для кэша и дешевле origin, но ломает смену URL и точный подсчёт кликов |
| **307/308** | семантически близки; 308 ≈ permanent как 301 — та же проблема контроля; не дают выигрыша для v1 |
| **200 + HTML meta refresh** | хуже для SEO/UX, лишняя сложность |

## References

- [DESIGN.sample.md §6 API, §11 Trade-offs](../DESIGN.sample.md)
