# CyberCity — конвенции общения микросервисов

Кросс-сервисный контракт: формат сообщений (event envelope) и общие правила
общения через брокер. Версионируется (`CONVENTIONS@vN`). Сервисы потребляют на
пиннённой версии; контракт выпущенной версии неизменен (новое — `@vN+1`).

> Скелет-источник — `<methodology-repo>/skeletons/hub/CONVENTIONS.md`; модель
> общения — `<methodology-repo>/docs/ARCHITECTURE.md`. Состав программы,
> реестр сервисов/интерфейсов/автономных компонентов и пинов — `COMPOSITION.md`.
> Широкие project-конвенции (иерархия доков, язык, ADR-формат, нейминг,
> логирование, коммиты, лицензии) — `AGENTS.md` (governance хаба).

## Версия

- **Текущая:** `CONVENTIONS@v1`
- **Совместимость:** backward-compatible (новое опц. поле) — без bump, остаться
  `@v1` (старые consumer'ы не ломаются). Breaking — major bump `@vN+1`.
- **Breaking →** `@v2` отдельным PR; сервисы мигрируют каждый своим PR (бамп пина
  + правки). Не атомарно — потому и нужен pin. См. `AGENTS.md` →
  *Версионирование контрактов*; `<methodology-repo>/docs/ARCHITECTURE.md`.

## Event envelope

Каноническая форма события, пересекающего границы сервисов через брокер
(`collector → Kafka → engine`; `data → Kafka → engine/manage`;
`manage → Kafka → engine`). Все сервисы публикуют/читают этот envelope.

```json
{
  "envelope_version": "1",
  "event_id": "<uuid>",
  "parent_event_ids": ["<uuid>"],
  "correlation_id": "<scenario/incident>",
  "tick": 0,
  "timestamp": "<RFC3339>",
  "source_type": "engine|service|scenario|player|system|collector",
  "source_id": "<id>",
  "event_type": "SCAN|ATTACK|COMPROMISE|city.build.completed|...",
  "target_id": "<topology node id | null>",
  "payload": {},
  "status": "pending|processed|failed|suppressed"
}
```

| Поле | Тип | Обяз. | Назначение |
|---|---|---|---|
| `envelope_version` | int | да | версия envelope (соответствует `CONVENTIONS@vN`; `1` для `@v1`) |
| `event_id` | uuid | да | идемпотентность (consumer дедуплирует по нему) |
| `parent_event_ids` | uuid[] | опц. | рёбра событийного графа (`caused_by`/`propagated_to`); причинность |
| `correlation_id` | string | опц. | сквозная трассировка сценария/инцидента |
| `tick` | int | да | логическое время tick-loop движка |
| `timestamp` | RFC3339 | да | когда событие произошло (не когда опубликовано) |
| `source_type` | enum | да | `engine \| service \| scenario \| player \| system \| collector` |
| `source_id` | string | да | идентификатор источника |
| `event_type` | string | да | тип события; пространство имён = `<domain>.<event>` (напр. `city.build.completed`, `host.scan`) или верхнеуровневый (`SCAN`, `COMPROMISE`) |
| `target_id` | string \| null | опц. | узел топологического графа — цель события |
| `payload` | object | да | данные события; схема по `event_type` |
| `status` | enum | да | `pending \| processed \| failed \| suppressed` |

События `cybercity-collector` дополнительно оборачиваются в **подписанный
конверт** (Ed25519) поверх этого envelope; Kafka — mTLS + ACL на продюсеров;
гости из range до брокера не достукиваются. Полный набор полей runtime-модели —
в [`cybercity-engine`/docs/MODELS.md](https://github.com/TheCipherKeeper/cybercity-engine/blob/main/docs/MODELS.md).

## Топики

- Именование: `<domain>.<event>` (например `city.build.completed`,
  `infra.provisioned`, `control.snapshot`). Верхнеуровневые имена (`SCAN`,
  `COMPROMISE`) допустимы для доменных событий collector'а.
- Сервис владеет топиками, которые публикует; consumer — подписывается.
- Реестр топиков по сервисам (publish/consume) — `COMPOSITION.md` → *Сервисы*;
  подробнее (назначение) — `ARCHITECTURE` сервисов.

## Правила

- Общение сервисов — **только через брокер**; прямые service-to-service вызовы
  (HTTP/RPC/общая БД между сервисами) в обход брокера запрещены. (Интерфейс →
  сервис по HTTP/WS presentation — разрешено; это клиентский край. Автономный компонент
  — не участник брокера, наблюдается out-of-band.)
- Не изобретать свой envelope в сервисах — только этот.
- Backward-compatible изменения (новое опц. поле) — **без bump**: остаться `@vN`.
  Breaking — **major bump** `@vN → @vN+1` отдельным PR. Схема целочисленная
  major-only (`@v1`, `@v2`, …) — minor-бампа нет.
- Idempotency: consumer дедуплирует по `event_id`.
- Логирование: structured JSON с `correlation_id` и `event_id` для сквозной
  трассировки; уровни `error`/`warn`/`info`/`debug`; секреты в логи никогда.

## ADR-ссылки

- [ADR-0003](adr/0003-collector-rust-out-of-band.md) — collector → Kafka (брокер Redpanda).
- [ADR-0004](adr/0004-runtime-kind-vm-container-lite.md) — движок-регистратор, не симулятор.
- [ADR-0008](adr/0008-topology-reachability-only-observed-propagation.md) — пропагация наблюдается.
- [ADR-0010](adr/0010-data-broker-producer.md) — data как broker-участник (`city.build.completed`).
