# ADR — CyberCity

Architecture Decision Records проекта. **Все ADR — и сквозные, и локальные —
живут здесь, в хабе `cybercity/adr/`; в репозиториях `docs/adr/` нет**
(см. [ADR-0005](0005-adr-centralized-in-hub.md)). Нумерация сквозная по всему
проекту; поле `Scope` показывает, чему относится решение.

| № | Scope | Решение | Статус |
|---|-------|---------|--------|
| [0001](0001-repo-composition.md) | Сквозное | Семь репозиториев (состав; `cybercity-clite` отдельным репо) | Accepted |
| [0002](0002-trust-boundary.md) | Сквозное | Доверенная vs best-effort плоскость | Accepted |
| [0003](0003-collector-rust-out-of-band.md) | Сквозное | Коллектор на Rust, out-of-band | Accepted |
| [0004](0004-runtime-kind-vm-container-lite.md) | Сквозное | `runtime_kind` {vm, container, lite}; движок — регистратор | Amended → 0006 |
| [0005](0005-adr-centralized-in-hub.md) | Сквозное | Все ADR — только в хабе `cybercity/adr/` | Accepted |
| [0006](0006-vulnerability-declarative-overlay-realism.md) | Сквозное | Уязвимость — декларативная сущность; overlay-артефакт; realism {real, narrative} | Accepted |
| [0007](0007-mvp-scope.md) | Сквозное | MVP scope — микро-город, container+lite, три зонда, append-only | Accepted |
| [0008](0008-topology-reachability-only-observed-propagation.md) | Сквозное | Топологический граф — только достижимость; пропагация наблюдается, не вычисляется | Accepted |
| [0009](0009-manage-implementation-language-go.md) | `manage` | Go как язык реализации контрольной плоскости (supersede стек-части ADR-0001) | Accepted |
| [0010](0010-data-broker-producer.md) | `data` | `cybercity-data` как broker-участник (publish `city.build.completed`); `CONVENTIONS@v1` | Accepted |

Формат ADR — в [`../AGENTS.md`](../AGENTS.md) → *ADR-формат*; шаблон —
[`_TEMPLATE.md`](_TEMPLATE.md). Состав и контракты — в
[`../COMPOSITION.md`](../COMPOSITION.md); envelope — в
[`../CONVENTIONS.md`](../CONVENTIONS.md).