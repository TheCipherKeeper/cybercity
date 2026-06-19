# ADR-0007: MVP scope — микро-город, container+lite, три зонда, append-only

## Status

Accepted

## Scope

Сквозное.

## Context

Целевая композиция ([`../COMPOSITION.md`](../COMPOSITION.md)) и модель уязвимостей
([ADR-0006](0006-vulnerability-declarative-overlay-realism.md)) описывают
полноценный полигон: 46 организаций / 263 сервиса / 464 линка, все три
`runtime_kind` (`vm`/`container`/`lite`), out-of-band-зонды полного каталога,
PostgreSQL-persistence, мульти-тенантность, авторскую DSL сценариев. Степень
реализации по репозиториям — разная: `cybercity-data` зрелый, остальное —
скелеты/стартовые точки (см.
[`../COMPOSITION.md`](../COMPOSITION.md) § «Статус реализации»).

Нужен минимальный релиз за 3 месяца (solo), который: (1) сохраняет
архитектурную идентичность и учебную сложность — языки (Python/Go/Rust/TS),
семь репозиториев, доверительную границу ([ADR-0002](0002-trust-boundary.md)),
out-of-band подписанный коллектор ([ADR-0003](0003-collector-rust-out-of-band.md)),
принцип «движок — регистратор, не симулятор» ([ADR-0004](0004-runtime-kind-vm-container-lite.md)),
overlay-модель vuln ([ADR-0006](0006-vulnerability-declarative-overlay-realism.md)),
два графа, replay; (2) режет только **объём и ширину**, не глубину и не
архитектуру; (3) честно помечает отложенное как «future», а не как
забытый долг.

## Decision

Минимальный релиз = **микро-город + одна kill-chain из 3 vuln + один
субстрат + три зонда + append-only лог + single-player**. Ни одна граница
[`../COMPOSITION.md`](../COMPOSITION.md) § «Кто чем владеет» не нарушена; режется
объём входа и набор включённых механизмов.

### 1. In-scope (в мин-релизе)

- **Город:** 3–4 организации, ~12 сервисов, ~25 линков. Модель, валидация,
  аллокатор IP/CIDR, `--seed` — без изменений; просто меньший вход в
  `config.yml`. Самый дешёвый выигрыш — аллокатор и build уже зрелые.
- **`runtime_kind`:** только `container` и `lite`. `vm` целиком за рамки
  мин-релиза (см. §2). Демонстрация покрывает и `narrative` (`clite`), и `real`
  (container overlay).
- **Субстрат развёртывания:** один K8s-кластер (kind/k3s). Cilium = сетевые
  политики (рёбра топологического графа), Multus = per-service IP (реальные
  сокеты, `nmap` видит город). Proxmox/ZFS/`vm` — отложены. `cybercity-manage`
  остаётся «control plane поверх IaC» — только IaC у́же (K8s-манифесты +
  сборка overlay-образов); overlay→manage→collector→engine цикл сохранён.
- **Зонды collector (MVP-каталог):** ровно три — `port_open`,
  `banner_match`, `canary_file`. Rust + Ed25519 + Kafka (Redpanda single-node)
  без изменений ([ADR-0003](0003-collector-rust-out-of-band.md)). Vuln-proof
  пишется только под эти зонды — словарь observables =这三个.
- **Persistence:** in-memory `WorldState` + append-only JSONL audit-log.
  Replay читает JSONL — детерминизм сохранён ([`../VISION.md`](../VISION.md)
  принцип «события — единственный источник истины»). PostgreSQL
  подключается позже как swap storage; движок не меняется.
- **Сценарии:** один YAML-сценарий — та kill-chain из 3 vuln (`preconditions`
  как рёбра DAG, `scoring.requires`). Контракт `data → engine` тот же;
  полная DSL авторства наращивается позже.
- **Vuln (3 сущности):** 1 `narrative` на `clite` (beachhead) + 2 `real` на
  `container` (lateral + финал). Доказывает overlay→`data build`→`overlays`-
  артефакт→`manage` (сборка образов)→`collector`→scoring end-to-end.
- **Мультиплейер:** single-player. Scoring-модель и так однопоточная
  (детерминированный `flag = f(seed, scenario_id, vuln_id)`,
  per-player-unique out of scope, [ADR-0006](0006-vulnerability-declarative-overlay-realism.md)
  §4). Демо в single-player = без рассинхрона с дизайном; scale-таблицу
  ([`../ARCHITECTURE.md`](../ARCHITECTURE.md) § «Целевые показатели») для
  MVP читаем как «1 игрок».
- **UI:** 2D-граф топологии (из `topology.json`) + live event-log (WebSocket
  от engine) + одно действие игрока. Покрывает критерии успеха
  ([`../VISION.md`](../VISION.md)) №1 (живой граф) и №2 (событие
  распространяется). Red/blue-дашборды, отчёты — future.

### 2. Deferred (явно за рамками мин-релиза; архитектурно заложено)

- **`vm` `runtime_kind` + Proxmox/ZFS** — самый тяжёлый кусок (ZFS
  snapshot/clone, образы VM, memory-introspection для `planted`-proof на VM).
  Архитектурно заложен в [ADR-0004](0004-runtime-kind-vm-container-lite.md);
  возвращается отдельным заходом при первом `real`-target на `vm`.
- **Зонды `process_spawn`/syscall/memory (Volatility)** — расширяют каталог;
  `planned`-proof на `vm` без них нереализуем out-of-band (подробно —
  future-док `cybercity-clite`/docs + расширение probe-каталога).
- **PostgreSQL** — swap storage поверх JSONL; persistence-контракт тот же.
- **Мультиплейер / атрибуция компрометации игрока / per-player-unique
  флаги** — production-концерн; противоречит replay ([ADR-0006](0006-vulnerability-declarative-overlay-realism.md)
  §4).
- **Полная DSL авторинга сценариев** — один YAML заменяется библиотекой;
  контракт `data → engine` не меняется.
- **Петли QA по статистике** (`difficulty` re-estimate, unintended-compromise
  как метрика) — требуют операционного трафика; на solo-демо
  aspirational, честно помечается, не выдаётся за работающее.

### 3. Обязательно закрыть на критическом пути демо

Эти пункты (открытые в предыдущих ADR/доках) — на пути мин-релиза, не
future:

- **Мини-спека `clite` `vuln_behavior` + probe-каталог**
  ([ADR-0006](0006-vulnerability-declarative-overlay-realism.md) §5) — без неё
  `narrative`/`lite` не runnable. Закрыть в первую неделю.
- **`schema_version` в event envelope** (см.
  [`../CONVENTIONS.md`](../CONVENTIONS.md) § «Event envelope») —
  cross-repo контракт эволюционирует; без поля версий — тихий рассинхрон.
- **Маппинг зонд → `vuln_id`** — при 3 vuln тривиален, но фиксируется как
  контракт (engine сопоставляет service-level compromise из коллектора с
  active vuln по probe_type).
- **[ADR-0004](0004-runtime-kind-vm-container-lite.md) → `Amended`** по
  `cve_id` (перенос в vuln-сущность, [ADR-0006](0006-vulnerability-declarative-overlay-realism.md)
  §1) — governance-гигиена: канонический текст не должен содержать
  устаревшее утверждение без маркера.

### 4. Дорожная карта мин-релиза (12 недель, solo)

- **Месяц 1 — фундамент:** закрыть clite/probe-контракт → collector MVP
  (Rust, Ed25519, Redpanda single-node, 3 зонда) → engine consumer-loop,
  in-memory state + JSONL audit, WebSocket API, минимальный tick-loop.
- **Месяц 2 — контент и деплой:** образ `clite` (Rust) → data: микро-город +
  3 манифеста vuln + overlay'и (1 clite-narrative + 2 container-real) +
  `overlays`-артефакт → manage на K8s (apply overlay→сборка→деплой pod'ов,
  reset = pod restart) → scoring rubric (3 vuln, kill-chain `requires`),
  один scenario-YAML, детерминированный флаг.
- **Месяц 3 — демо и полировка:** UI (граф + event-log + действие игрока) →
  end-to-end на kind/k3s (Cilium policies, Multus IP, replay из JSONL) →
  hardening, детерминизм, синк доков → публичный read-only демо (Cloudflare
  tunnel), запись walkthrough.

## Consequences

### Positive

- Архитектурная идентичность сохранена полностью; режется объём входа и
  набор включённых механизмов. Учебная сложность (доверительная граница,
  out-of-band подписанный поток, два графа, overlay-модель vuln, hybrid
  execution, replay, 4 языка / 7 репо) — на месте.
- Демонстрируется end-to-end overlay→collector→scoring на `container`, т.е.
  «настоящая» часть пайплайна, а не только дешёвый фон.
- Дрейф отложенного к «забытому» предотвращён явным списком §2.

### Negative

- `vm`-`real` (самый ценный hands-on случай) не входит в демо — `real`
  демонстрируется только на `container`. Смягчение: контракт `vm` заложен,
  возвращается без переписывания архитектуры.
- `planted`-proof на `vm` (memory/fs-introspection) не реализован; `real`
  на `container` использует `canary_file`/`banner_match` из 3-зондового
  набора.
- Single-player: scale-таблица «10 игроков» для MVP не подтверждена
  моделью — правится на «1 игрок, MVP».

## Alternatives considered

- **Оставить Proxmox/`vm` в мин-релизе:** отвергнуто — ZFS, образы VM и
  memory-introspection для `planted`-proof поглощают 3 месяца ради
  runtime-вида, который демо не требует; K8s-only даёт реальную сеть
  (Cilium+Multus) при доле сложности.
- **Режь по языкам/архитектуре:** отвергнуто — противоречит учебной цели
  проекта (намеренно сложный); режем объём и ширину, не глубину.
- **Урезать до `lite`-only (без `real`):** отвергнуто — тогда не доказан
  overlay→manage→сборка→collector цикл для настоящих багов; `real` на
  `container` остаётся как showpiece пайплайна.
- **PostgreSQL сразу:** отвергнуто на MVP — append-only JSONL даёт replay и
  what-if без DB; PostgreSQL = swap storage позже, движок не меняется.

## Related

- [`../COMPOSITION.md`](../COMPOSITION.md) — состав, контракты, статус
  реализации (правка scale на single-player MVP).
- [`../ARCHITECTURE.md`](../ARCHITECTURE.md) — дорожная карта к первой
  демонстрации; § «Целевые показатели масштабируемости» (правка на MVP).
- [`../VISION.md`](../VISION.md) — критерии успеха (№1, №2 покрываются UI
  мин-релиза).
- [ADR-0004](0004-runtime-kind-vm-container-lite.md) — `runtime_kind`; `vm`
  отложен, `container`/`lite` в MVP.
- [ADR-0006](0006-vulnerability-declarative-overlay-realism.md) — overlay-
  модель; `narrative`+`real` в MVP на `lite`/`container`; `vm`-`real` future.