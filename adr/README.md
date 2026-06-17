# ADR — CyberCity (сквозные решения)

Architecture Decision Records, затрагивающие несколько репозиториев.
Локальные решения каждого репозитория — в его собственном `docs/adr/`.

| № | Решение | Статус |
|---|---------|--------|
| [0001](0001-six-repo-composition.md) | Разделение на 6 репозиториев | Accepted |
| [0002](0002-trust-boundary.md) | Доверенная vs best-effort плоскость | Accepted |
| [0003](0003-collector-rust-out-of-band.md) | Коллектор на Rust, out-of-band | Accepted |
| [0004](0004-runtime-kind-vm-container-lite.md) | `runtime_kind` {vm, container, lite} + `honeypot`-назначение; движок — регистратор | Accepted |

Формат ADR — в [`../CONVENTIONS.md`](../CONVENTIONS.md). Состав и контракты —
в [`../COMPOSITION.md`](../COMPOSITION.md).