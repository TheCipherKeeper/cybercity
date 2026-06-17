# ADR-0003: Коллектор на Rust, out-of-band

## Status

Accepted

## Context

Нужен наблюдатель за гостями, поток которого можно доверять (см. ADR-0002).
In-guest «агент» (`cybercity-agents` / `cybercity-node-agent`) ненадёжен: живёт
в компрометируемой среде, может быть подменён, тянуть брокера из гостя нельзя.

## Decision

- **Out-of-band, не in-guest:** коллектор работает на гипервизоре/K8s-узле,
  наблюдает гостей снаружи (ZFS-snapshot/overlay2-walk, eBPF/Zeek,
  `virsh dump-guest-memory`/Volatility, `/proc`-walk, Falco/Tetragon),
  read-only.
- **Rust:** системный язык, низкие накладные расходы на зонд, строгая
  типизация, безопасная работа с памятью — подходит для per-host демона,
  которому доверяем.
- Подпись событий Ed25519; транспорт Kafka (mTLS + ACL); control-канал от
  `cybercity-manage`.
- Переименование: `cybercity-agents` → `cybercity-collector`; crate-имена
  `ccna-*` → `ccc-*`; бинарник `cybercity-node-agent` → `cybercity-collector`.

## Consequences

### Positive

- Поток, которому можно доверять для scoring (вместе с ADR-0002).
- Зонды не зависят от гостевой ОС и не светятся в range.

### Negative

- Зонды требуют прав на гипервизоре/узле; часть — привилегированные.
- Состояние кода сейчас — in-guest MVP; переход на out-of-band — отдельный заход.

## Alternatives considered

- **In-guest агент (как было)**: отвергнут — ненадёжен в скомпрометированной среде.
- **Go/Python для коллектора**: возможны, но Rust даёт лучший профиль для
  per-host демона с доверием.

## Related

- [`0002-trust-boundary.md`](0002-trust-boundary.md).
- [`../COMPOSITION.md`](../COMPOSITION.md) — «История переименований».