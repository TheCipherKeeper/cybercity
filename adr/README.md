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
| [0004](0004-runtime-kind-vm-container-lite.md) | Сквозное | `runtime_kind` {vm, container, lite}; движок — регистратор | Accepted |
| [0005](0005-adr-centralized-in-hub.md) | Сквозное | Все ADR — только в хабе `cybercity/adr/` | Accepted |

Формат ADR — в [`../CONVENTIONS.md`](../CONVENTIONS.md). Состав и контракты —
в [`../COMPOSITION.md`](../COMPOSITION.md).