# CyberCity — композиция

Канонический источник правды о составе проекта. Все репозитории
ссылаются сюда; их README держат только короткую сводку + ссылку на
этот файл.

**CyberCity** — модульный кибер-полигон: цифровой двойник города
(IT/OT) для учений red/blue. Каждый репозиторий — один слайс системы;
одна и та же декларативная модель города, разные слои. Всё крутится
на вашем Proxmox / K8s, никакого SaaS, никакой внешней телеметрии.

> Это **целевая** композиция. Степень реализации разная по репозиториям
> (см. «Статус реализации» внизу); документ фиксирует договорённости,
> к которым идёт код, а не только то, что уже построено.

## Документация в этом репозитории

Помимо этого файла, хаб держит системные документы:

- [`VISION.md`](VISION.md) — философия, принципы, аудитории, критерии успеха.
- [`ARCHITECTURE.md`](ARCHITECTURE.md) — системная архитектура и контекст.
- [`CONVENTIONS.md`](CONVENTIONS.md) — кросс-репо конвенции, иерархия документов,
  скелет репозитория, форматы ADR/README, event envelope.
- [`adr/`](adr/) — сквозные архитектурные решения (почему 6 репо, доверительная
  граница, Rust-коллектор).

Каждый репозиторий `cybercity-*` держит собственный `AGENTS.md` (governance) и
`docs/` (как реализовано в нём) и ссылается сюда как к канону.

## Репозитории

| Слой | Репо | Язык | Назначение |
|---|---|---|---|
| Витрина | [`cybercity`](https://github.com/TheCipherKeeper/cybercity) | — | обложка/индекс проекта; канон композиции (этот файл) |
| Данные | [`cybercity-data`](https://github.com/TheCipherKeeper/cybercity-data) | Python | декларативная модель города (source of truth) + авторинг сценариев |
| Runtime | [`cybercity-engine`](https://github.com/TheCipherKeeper/cybercity-engine) | Go | событийное ядро: топологический + причинный граф, tick-loop, replay, эмуляция трафика, scoring |
| Управление | [`cybercity-manage`](https://github.com/TheCipherKeeper/cybercity-manage) | Python | контрольная плоскость: provisioning, reset/rollback, изоляция, квоты, мульти-тенантность; оркестрирует Proxmox API + Terraform/Pulumi; размещает доверенный коллектор |
| Коллектор | [`cybercity-collector`](https://github.com/TheCipherKeeper/cybercity-collector) | Rust | внешний out-of-band per-host коллектор: зонды (fs/net/mem/proc/syscall), подписанные события в engine по Kafka; недосягаем из range-сегмента |
| Визуал | [`cybercity-ui`](https://github.com/TheCipherKeeper/cybercity-ui) | TS (+возм. Rust) | 2D-карта топологии, таймлайн событий, дашборды red/blue, отчёты |

## Поток данных и контракты

1. `cybercity-data build` → `engine.zip` (внутри `runtime/engine.json`,
   `topology.json`, `attack-surface.json`, `schema.json`) — контракт
   **data → engine/ui**. Модель города: 46 организаций / 263 сервиса /
   464 линка; IP/CIDR генерируются аллокатором, воспроизводимо через `--seed`.
2. `cybercity-data` также авторит **сценарии** → артефакт сценария
   (цели, injects, флаги, scoring-rubric, timebox) — контракт
   **data → engine**. `data` порождает декларацию, `engine` исполняет.
3. `cybercity-engine` грузит `engine.zip`, ведёт world-state и причинный
   граф, исполняет сценарии, считает scoring.
4. `cybercity-collector` (по одному на хост) наблюдает гостей **снаружи** →
   подписанные события по Kafka (mgmt-плоскость) → `cybercity-engine` как
   **авторитетный** поток (на нём считается scoring); control-канал идёт
   от `cybercity-manage` («наблюдать X», «снапшот сейчас», «обновить политику»).
5. `cybercity-manage` дёргает гипервизор/фабрику: provisioning, snapshot/reset,
   изоляция; `engine` слышит об изменениях инфры как о смене сим-состояния.
6. `cybercity-ui` читает `topology.json` + поток событий `engine` (WebSocket).

## Доверительная граница

- **Доверенная плоскость (trusted):** `cybercity-manage` +
  `cybercity-collector` + Kafka-брокер — живут в mgmt-сегменте, **без
  маршрута из range**. На их потоке считается scoring.
- **Ненадёжная плоскость (best-effort):** всё внутри гостевых VM/контейнеров
  (включая опциональный in-guest enrichment) — **никогда** не источник
  для scoring.
- `cybercity-collector` подписывает события (Ed25519); Kafka — mTLS + ACL
  на продюсеров; гости до брокера не достукиваются структурно.

## Кто чем владеет (границы ответственности)

- **Provisioning / reset / изоляция на уровне инфры** → `manage`
  (гипервизор/фабрика). `engine` только *слышит* об этом как о смене
  состояния.
- **События / причинность / replay / scoring-логика** → `engine`.
- **Декларация мира и сценариев** → `data`. `engine` *исполняет* сценарий,
  не авторит.
- **Наблюдение снаружи** → `collector`. **Действие над гостем** (reset/
  изоляция) — через `manage`/фабрику, **не** через in-guest агент.
  In-guest enrichment — опционально, best-effort.

## История переименований

- `cybercity-agents` → **`cybercity-collector`** (Rust): переосмыслен из
  in-guest «агента» во внешний out-of-band per-host коллектор. Crate-имена
  `ccna-*` → `ccc-*` (cyber city collector); бинарник `cybercity-node-agent`
  → `cybercity-collector`.
- `cybercity-blueprints` → **`cybercity-manage`** (Python): из IaC-шаблонов
  (Ansible/Terraform) в контрольную плоскость, оркестрирующую реальный IaC
  под собой (а не переписывающую provisioning заново).
- `cybercity-scenarios` (упоминался в старых доках) — **не отдельное репо**;
  авторинг сценариев вошёл в `cybercity-data`.
- `cybercity-simulator` (упоминался в старых доках) — **не отдельное репо**;
  эмуляция трафика вошла в `cybercity-engine`.

## Статус реализации (кратко)

- `cybercity-data` — зрелый: модель, валидация, аллокатор, сборка артефактов,
  CI (95% coverage, mypy --strict). Авторинг сценариев — в работе.
- `cybercity-engine` — скелет: домен, tick-loop, причинный граф, API/WS.
  TODO: persistence (PostgreSQL), consumer-loop Redpanda, reset-from-snapshot,
  runner сценариев, scoring.
- `cybercity-collector` — MVP-скелет: текущий код ещё in-guest (config/
  host-bridge/telemetry/kafka-transport/command). Рефакторинг зондов в
  out-of-band (fs/net/mem/proc/syscall) + настоящая Ed25519-подпись + реальный
  Kafka-transport — отдельный заход.
- `cybercity-manage` — стартовая точка: контрольная плоскость поверх
  Proxmox API + Terraform/Pulumi (provisioning, reset, изоляция, квоты).
- `cybercity-ui` — каркас: карта, таймлайн, дашборды.

## Лицензии

- Код: MIT. Документация: CC BY 4.0.