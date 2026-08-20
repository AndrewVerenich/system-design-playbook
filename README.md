# System Design Playbook

Коллекция архитектурных разборов крупных систем в формате System Design interview.  
Репозиторий демонстрирует практику проектирования: требования, оценка нагрузки, high-level архитектура, модель данных, API, масштабирование, отказоустойчивость и явные trade-offs.

Каждый раунд — законченный design-документ (ADR/RFC-стиль), пригодный и для подготовки к интервью, и как материал портфолио.

## Содержание: раунды

Краткий список — ссылки ведут к полному описанию в [разделе ниже](#rounds-overview).

0. [url-shortener](#0-url-shortener) — сокращение ссылок, redirect hot path, генерация кода, cache + KV storage *(образец)*  
1. *(следующие раунды: rate limiter, pastebin, notification service, chat, …)*

---

## Как проходить раунд

1. **Brief** — задача и ограничения (как на интервью).
2. **Draft** — свой `DESIGN.md` + схемы (Miro PNG и/или Mermaid).
3. **Review** — разбор: сильные стороны, дыры, follow-up вопросы.
4. **Revise** — правка до accepted-версии.

Правило: на новых раундах сначала свой вариант, потом разбор.  
**Round 00** содержит полный **образец** (`DESIGN.sample.md`) — ориентир по глубине и структуре.

## Структура репозитория

```text
system-design-playbook/
├── README.md
├── _template/
│   └── DESIGN.md
└── rounds/
    └── 00-url-shortener/
        ├── BRIEF.md
        ├── DESIGN.sample.md   ← образец полного разбора
        ├── diagrams/
        ├── DESIGN.md          ← твой драфт (на новых раундах)
        └── REVIEW.md
```

Шаблон: [`_template/DESIGN.md`](./_template/DESIGN.md).

---

<a id="rounds-overview"></a>

## Раунды

### 0. [url-shortener](./rounds/00-url-shortener/BRIEF.md)

Проектирование сервиса сокращения URL (TinyURL / bit.ly-like) с упором на read-heavy redirect path.

**Описание:**
- Clarifying questions, FR/NFR, back-of-the-envelope расчёты.
- High-level архитектура: Write API / Redirect API / cache / primary store.
- Генерация short code (Snowflake → Base62), custom alias, коллизии.
- Cache-aside + защита от stampede, 301 vs 302.
- Масштабирование, failure modes, явные trade-offs.
- Mermaid-диаграммы: контекст, write/read path, data model.

**Артефакты:**
- [BRIEF.md](./rounds/00-url-shortener/BRIEF.md) — условие раунда
- [DESIGN.sample.md](./rounds/00-url-shortener/DESIGN.sample.md) — **образец полного дизайна** (RFC / design doc стиль)
- [adr/](./rounds/00-url-shortener/adr/) — примеры **ADR** по ключевым решениям
- [REVIEW.md](./rounds/00-url-shortener/REVIEW.md) — образец review
- [diagrams/](./rounds/00-url-shortener/diagrams/) — схемы

**Фокус компетенций:**
- Capacity estimation
- KV access pattern + caching
- Unique ID generation
- Hot key / viral traffic
- Consistency vs latency на redirect path
