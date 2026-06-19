# CyberCity — композиция

Канонический источник правды о составе проекта: репозитории, контракты,
доверительная граница, ownership, имена, статус реализации. Все репозитории
ссылаются сюда; их README держат только короткую сводку + ссылку на этот файл.

**CyberCity** — модульный кибер-полигон: цифровой двойник города
(IT/OT) для учений red/blue. Каждый репозиторий — один слайс системы;
одна и та же декларативная модель города, разные слои. Всё крутится
на вашем Proxmox / K8s, никакого SaaS, никакой внешней телеметрии.

> Это **целевая** композиция. Степень реализации разная по репозиториям
> (см. «Статус реализации» внизу); документ фиксирует договорённости,
> к которым идёт код, а не только то, что уже построено.

## Репозитории

| Слой | Репо | Язык | Назначение |
|---|---|---|---|
| Витрина | [`cybercity`](https://github.com/TheCipherKeeper/cybercity) | — | обложка/индекс проекта; системные документы (без кода) |
| Данные | [`cybercity-data`](https://github.com/TheCipherKeeper/cybercity-data) | Python | декларативная модель города (source of truth) + авторинг сценариев + уязвимости (манифест + overlay-исходники) |
| Runtime | [`cybercity-engine`](https://github.com/TheCipherKeeper/cybercity-engine) | Go | событийное ядро: топологический + причинный граф, tick-loop, replay, эмуляция трафика, scoring |
| Lite-цель | [`cybercity-clite`](https://github.com/TheCipherKeeper/cybercity-clite) | Rust | параметризуемый stub-образ `clite` для `runtime_kind: lite` (реальный сокет + поддельный баннер по дескриптору сервиса); живёт в range-сегменте, наблюдается коллектором out-of-band |
| Управление | [`cybercity-manage`](https://github.com/TheCipherKeeper/cybercity-manage) | Python | контрольная плоскость: provisioning, reset/rollback, изоляция, квоты, мульти-тенантность; оркестрирует Proxmox API + Terraform/Pulumi; размещает доверенный коллектор; generic consumer `overlays`-артефакта (сборка образов по service-mapping) |
| Коллектор | [`cybercity-collector`](https://github.com/TheCipherKeeper/cybercity-collector) | Rust | внешний out-of-band per-host коллектор: зонды (fs/net/mem/proc/syscall), подписанные события в engine по Kafka; недосягаем из range-сегмента |
| Визуал | [`cybercity-ui`](https://github.com/TheCipherKeeper/cybercity-ui) | TS (+возм. Rust) | 2D-карта топологии, таймлайн событий, дашборды red/blue, отчёты |

## Поток данных и контракты

1. `cybercity-data build` → `engine.zip` (внутри `runtime/engine.json`,
   `topology.json`, `attack-surface.json`, `schema.json`) — контракт
   **data → engine/ui**. Модель города: 46 организаций / 263 сервиса /
   464 линка; IP/CIDR генерируются аллокатором, воспроизводимо через `--seed`.
2. `cybercity-data build` → **`overlays`-артефакт** (каталог уязвимостей +
   tarball overlay-плейбуков) — контракт **data → manage**
   ([ADR-0006](adr/0006-vulnerability-declarative-overlay-realism.md)).
   Уязвимость — first-class сущность: манифест + overlay-исходники рядом (один
   PR = одна vuln); `cve_id` живёт в vuln-сущности, не в дескрипторе сервиса.
3. `cybercity-data` также авторит **сценарии** → артефакт сценария
   (цели, injects, флаги, scoring-rubric, timebox) — контракт
   **data → engine**. `data` порождает декларацию, `engine` исполняет.
4. `cybercity-engine` грузит `engine.zip`, ведёт world-state и причинный
   граф, исполняет сценарии, считает scoring.
5. `cybercity-collector` (по одному на хост) наблюдает гостей **снаружи** →
   подписанные события по Kafka (mgmt-плоскость) → `cybercity-engine` как
   **авторитетный** поток (на нём считается scoring); control-канал идёт
   от `cybercity-manage` («наблюдать X», «снапшот сейчас», «обновить политику»).
6. `cybercity-manage` — **generic consumer** `overlays`-артефакта: по
   `service-mapping` + `overlay-id` собирает образы (Packer/Ansible) и деплоит;
   дёргает гипервизор/фабрику (provisioning, snapshot/reset, изоляция). Семантики
   vuln не знает. `engine` слышит об изменениях инфры как о смене сим-состояния.
7. `cybercity-ui` читает `topology.json` + поток событий `engine` (WebSocket).

## Доверительная граница

- **Доверенная плоскость (trusted):** `cybercity-manage` +
  `cybercity-collector` + Kafka-брокер — живут в mgmt-сегменте, **без
  маршрута из range**. На их потоке считается scoring.
- **Ненадёжная плоскость (best-effort):** всё внутри гостевых VM/контейнеров
  (включая опциональный in-guest enrichment) — **никогда** не источник
  для scoring.
- `cybercity-collector` подписывает события (Ed25519); Kafka — mTLS + ACL
  на продюсеров; гости до брокера не достукиваются структурно.
- Все runtime-цели (`vm` / `container` / `lite`) наблюдаются коллектором
  out-of-band единообразно; класса «engine-synthesized service events» нет —
  движок регистратор, не симулятор (см.
  [`adr/0004-runtime-kind-vm-container-lite.md`](adr/0004-runtime-kind-vm-container-lite.md)).

Обоснование — в [`adr/0002-trust-boundary.md`](adr/0002-trust-boundary.md).

## Кто чем владеет (границы ответственности)

- **Provisioning / reset / изоляция на уровне инфры** → `manage`
  (гипервизор/фабрика). `engine` только *слышит* об этом как о смене
  состояния.
- **События / причинность / replay / scoring-логика** → `engine`.
- **World-state / persistence (PostgreSQL)** → `engine` и только `engine`.
  `engine` — единственный читатель и писатель PostgreSQL (снапшоты
  `WorldState` + audit log). **UI и manage в БД не ходят.** UI читает
  статичную `topology.json` (из `data`) + live-поток `engine` по WebSocket;
  manage координирует `engine` через control API и Redpanda, но world-state
  не владеет.
- **manage ↔ engine** — два канала: control API `manage → engine`
  (HTTP/gRPC: старт/пауза/сброс сценария, запрос снапшота, reload
  топологии) + Redpanda control-topic `manage → engine` (уведомления об
  изменениях инфры: provisioning/reset/изоляция — engine слышит как смену
  сим-состояния).
- **Декларация мира, сценариев и уязвимостей** → `data`. Уязвимость —
  first-class сущность (манифест + overlay-исходники, `realism ∈ {real,
  narrative}`); `cve_id` живёт в vuln-сущности, не в дескрипторе сервиса
  ([ADR-0006](adr/0006-vulnerability-declarative-overlay-realism.md)). `engine`
  *исполняет* сценарий, не авторит; `manage` *собирает* образы из
  `overlays`-артефакта, не владеет контентом vuln.
- **Наблюдение снаружи** → `collector`. **Действие над гостем** (reset/
  изоляция) — через `manage`/фабрику, **не** через in-guest агент.
  In-guest enrichment — опционально, best-effort.
- **`runtime_kind`** (`vm`/`container`/`lite`, deployment-time) → `manage`
  (service-mapping manifest). Движок — **регистратор**, не симулятор. См.
  [`adr/0004-runtime-kind-vm-container-lite.md`](adr/0004-runtime-kind-vm-container-lite.md).
- **Образ `lite`-цели** (`clite`) → `cybercity-clite` (Rust): параметризуется
  дескриптором сервиса, деплоится в range-сегмент, наблюдается `collector`
  out-of-band. См. [`adr/0001-repo-composition.md`](adr/0001-repo-composition.md)
  и [`adr/0004-runtime-kind-vm-container-lite.md`](adr/0004-runtime-kind-vm-container-lite.md).

## Имена и обоснование

- **`cybercity-collector`** (Rust) назван коллектором, а не «агентом», потому что
  это внешний out-of-band per-host наблюдатель, а не in-guest агент
  (ADR-0003). Crate-имена `ccc-*` (cyber city collector); бинарник
  `cybercity-collector`.
- **`cybercity-manage`** (Python) — контрольная плоскость, оркестрирующая реальный
  IaC (Ansible/Terraform/Pulumi) под собой, а не переписывающая provisioning
  заново.
- **`cybercity-clite`** (не `cybercity-lite`), чтобы не читалось как «облегчённая
  cybercity»; образ/бинарь — `clite` (от «container lite»). См. ADR-0001/0004.
- Авторинг сценариев живёт **в `cybercity-data`**, отдельного репо сценариев нет.
- Эмуляция трафика живёт **в `cybercity-engine`**, отдельного репо симулятора нет.

## Статус реализации (кратко)

- `cybercity-data` — зрелый: модель, валидация, аллокатор, сборка артефактов,
  CI (95% coverage, mypy --strict). Авторинг сценариев — в работе. Авторинг
  уязвимостей (манифест + overlay-исходники, ADR-0006) — не начат.
- `cybercity-engine` — скелет: домен, tick-loop, причинный граф, API/WS.
  TODO: persistence (PostgreSQL), consumer-loop Redpanda, reset-from-snapshot,
  runner сценариев, scoring.
- `cybercity-collector` — стартовый скелет (config/transport/command). Целевой
  облик — out-of-band зонды (fs/net/mem/proc/syscall) с настоящей
  Ed25519-подписью и реальным Kafka-transport; доведение до него — отдельный
  заход (ADR-0003).
- `cybercity-manage` — стартовая точка: контрольная плоскость поверх
  Proxmox API + Terraform/Pulumi (provisioning, reset, изоляция, квоты).
- `cybercity-clite` — стартовая точка: образ `clite` — параметризуемая
  заглушка (биндит порты, поддельный баннер по дескриптору, heartbeat
  коллектору). Контракт дескриптор → поведение и `clite` ↔ collector — TBD;
  для `narrative`-vuln (`vuln_behavior`: trigger → materialize-observable)
  решение зафиксировано в
  [ADR-0006](adr/0006-vulnerability-declarative-overlay-realism.md), полная
  мини-спека — будущий док `cybercity-clite`/docs.
- `cybercity-ui` — каркас: карта, таймлайн, дашборды.

Дорожная карта к первой публичной демонстрации — в
[`ARCHITECTURE.md`](ARCHITECTURE.md) (§ «Дорожная карта»).

## Связанные документы

- [`VISION.md`](VISION.md) — зачем проект существует, принципы, аудитории.
- [`ARCHITECTURE.md`](ARCHITECTURE.md) — системная архитектура, два графа, hybrid execution, сетевая топология, слои.
- [`DATA_FLOW.md`](DATA_FLOW.md) — runtime-динамика: жизненный цикл события и сценария, end-to-end потоки, replay/scoring.
- [`CONVENTIONS.md`](CONVENTIONS.md) — кросс-репо конвенции и правило лицензий.
- [`adr/`](adr/) — сквозные архитектурные решения.