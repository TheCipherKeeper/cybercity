# CyberCity — Архитектура

Системный взгляд: как слои связаны, где что живёт, какие контракты между
ними. Внутреннее устройство каждого репозитория — в его собственном
`docs/ARCHITECTURE.md`; состав и доверительная граница — в
[`COMPOSITION.md`](COMPOSITION.md).

## Системный контекст

```text
┌─────────────────────────────────────────────────────────────────────┐
│                         Внешние пользователи                         │
│   Игроки │ Инструкторы │ Read-only посетители │ Авторы сценариев    │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Платформа CyberCity                          │
│                                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────┐  │
│  │     UI      │    │   Engine    │    │  cybercity-data         │  │
│  │  (React/    │◄──►│    (Go)     │◄──►│  (модель + сценарии,    │  │
│  │  WebSocket) │    │             │    │   Python)               │  │
│  └──────┬──────┘    └──────┬──────┘    └─────────────────────────┘  │
│         │                   │                                        │
│         │                   ▼                                        │
│         │          ┌─────────────────┐                               │
│         │          │ Redpanda/Kafka  │◄──── cybercity-collector      │
│         │          │  (event bus)   │      (Rust, out-of-band,      │
│         │          └─────────────────┘       подписанные события)    │
│         │                   │                  ▲                    │
│         │     ┌─────────────┼─────────────┐    │ control: manage     │
│         │     ▼             ▼             ▼                            │
│  ┌──────▼─────┐   ┌────────▼────────┐   ┌──────────────┐            │
│  │ PostgreSQL │   │  Real services  │   │ lite stubs   │            │
│  │  (state)   │   │  (VM / pod)     │   │ (cc-lite)    │            │
│  └────────────┘   └─────────────────┘   └──────────────┘            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Инфраструктурный слой: cybercity-manage (контрольная плоскость)     │
│  поверх Proxmox + Kubernetes + Cilium + Multus                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Основные ответственности

| Компонент | Ответственность |
|-----------|-----------------|
| **cybercity-data** | Декларативная модель города (source of truth), валидация, генерация артефактов (`engine.zip`, `topology.json`, …), авторинг сценариев. |
| **cybercity-engine** | Runtime-состояние, обработка событий, propagation, причинный граф, снапшоты, scoring, исполнение сценариев. **Единственный мутатор состояния.** |
| **cybercity-ui** | Визуализация (карта, таймлайн, дашборды), ввод игрока, real-time обновления по WebSocket. |
| **cybercity-manage** | Контрольная плоскость: provisioning, reset/rollback, изоляция, квоты/мульти-тенантность; размещает коллектор на хостах. |
| **cybercity-collector** | Внешний out-of-band per-host наблюдатель; подписанные события в engine по Kafka; control-канал от manage. |
| **Redpanda / Kafka** | Event bus между engine, ui, коллектором, реальными сервисами. |
| **PostgreSQL** | Снапшоты `WorldState` и audit log событийного графа. |
| **MinIO / S3** | Артефакты `engine.zip` и replay-дампы. |

## Два графа — модель города

Город моделируется через два связанных графа (концепция; поля — в
`cybercity-engine`/docs/MODELS.md):

- **Топологический граф** (статический) — *что с чем связано*. Загружается из
  `cybercity-data`, иммутабелен во время симуляции. Узлы — сервисы, рёбра —
  декларированные связи (`api-call`, `auth`, `db-read`, `backup-of`, …) плюс
  inferred (`same_network`, `same_org`, `exposure_chain`).
- **Событийный граф** (динамический, append-only) — *что произошло и почему*.
  Узлы — события, рёбра — `caused_by`, `propagated_to`, `triggered_rule`,
  `response_to`. Даёт attack provenance, replay, explainability.

Топология — рельсы; события — поезда. Подробно — в
[`cybercity-engine`/docs/ARCHITECTURE.md](https://github.com/TheCipherKeeper/cybercity-engine/blob/main/docs/ARCHITECTURE.md).

## Hybrid execution

| `runtime_kind` | Что это | Кто отвечает на события | Когда используется |
|-------|---|---|----|
| **vm** | полная VM, real OS/software | Out-of-band наблюдатель (`cybercity-collector`) | High-value target, Windows/OT, persistence |
| **container** | контейнер, real software (gVisor/Kata) | Out-of-band наблюдатель | real-сервис на shared-ядре, плотнее VM |
| **lite** | лёгкий stub-контейнер: реальный сокет + подделанный баннер | Out-of-band наблюдатель | Массовый фон города (замена «simulated») |

`runtime_kind` — deployment-time concern, не часть канонической city data
(назначается в `cybercity-manage` service-mapping manifest). По умолчанию —
`lite`. `honeypot` — отдельный флаг назначения-наживки (бывший `decoy`),
ортогонален `runtime_kind` (honeypot может быть `lite` или `vm`). Все runtime-цели
наблюдаются коллектором единообразно; движок — регистратор, не симулятор
(класса «engine-synthesized service events» нет). Обоснование —
[`adr/0004-runtime-kind-vm-container-lite.md`](adr/0004-runtime-kind-vm-container-lite.md).

## Слои развёртывания

| Слой | Назначение | Примеры инструментов |
|------|------------|----------------------|
| **Management** | Админский доступ, CI/CD, мониторинг; живут manage + коллектор + Kafka | Proxmox host, Terraform, Ansible |
| **Control** | Движок, БД, messaging, GitOps | K8s, Redpanda, PostgreSQL, ArgoCD |
| **City / Data** | Real VMs, lite stub-контейнеры, player workstations | VMs, Multus, Cilium, VyOS |

## Observability

- **Метрики:** tick duration, queue depth, event throughput, health сервисов.
- **Логи:** structured JSON с `correlation_id` и `event_id`.
- **Трейсы:** lineage событий через событийный граф.
- **Дашборды:** Grafana с city-level и per-service видами.

## Модель безопасности

- Сетевая сегментация явно задана в топологическом графе.
- Публичные сервисы достижимы только через declared exposure.
- OT-сегменты изолированы от management и публичных сетей.
- Наблюдение за гостями — только out-of-band (`cybercity-collector`),
  подписанные события; in-guest телеметрия — best-effort, не для scoring.
- Публичный UI read-only; действия игрока требуют аутентифицированной сессии.
- Секреты в Vault или cloud KMS, никогда в репозиториях.

См. [ADR-0002: доверительная граница](adr/0002-trust-boundary.md).

## Целевые показатели масштабируемости

| Ресурс | Home lab | Production-набросок |
|--------|----------|---------------------|
| Сервисы | 300 | 1,000+ |
| Событий/сек | 100 | 10,000+ |
| Игроки | 10 | 100+ |
| Real VMs/контейнеры | 6–10 | 50–100 |
| Latency | <1s на tick | <100ms на событие |

## Дорожная карта к первой публичной демонстрации

1. **Core engine** ✅ — topology, event graph, router, state, API.
2. **Persistence** — PostgreSQL-снапшоты и audit.
3. **Messaging** — интеграция с Redpanda.
4. **Scenario runner** — первый скриптованный сценарий (авторинг в data).
5. **UI** — интерактивный граф, event log, панель команд.
6. **Home lab deployment** — Proxmox + K8s через manage.
7. **Public read-only demo** — Cloudflare tunnel.

## Связанные документы

- [`COMPOSITION.md`](COMPOSITION.md) — состав, контракты, доверительная граница.
- [`VISION.md`](VISION.md) — философия и принципы.
- [`CONVENTIONS.md`](CONVENTIONS.md) — кросс-репо конвенции и иерархия документов.
- [`adr/`](adr/) — сквозные архитектурные решения.
- [`cybercity-engine`/docs/ARCHITECTURE.md](https://github.com/TheCipherKeeper/cybercity-engine/blob/main/docs/ARCHITECTURE.md) — внутреннее устройство движка.
- [`cybercity-data`/docs/ARCHITECTURE.md](https://github.com/TheCipherKeeper/cybercity-data/blob/main/docs/ARCHITECTURE.md) — внутреннее устройство data.