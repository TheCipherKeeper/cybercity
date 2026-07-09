# ADR-0003: Коллектор на Rust, out-of-band

## Status

Accepted

## Scope

Сквозное.

## Context

Нужен наблюдатель за гостями, поток которого можно доверять (см. ADR-0002).
In-guest агент ненадёжен по определению: живёт в компрометируемой среде, может
быть подменён, тянуть брокера из гостя нельзя.

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
- Crate-имена `ccc-*` (cyber city collector); бинарник `cybercity-collector`.

## Consequences

### Positive

- Поток, которому можно доверять для scoring (вместе с ADR-0002).
- Зонды не зависят от гостевой ОС и не светятся в range.

### Negative

- Зонды требуют прав на гипервизоре/узле; часть — привилегированные.
- Построение out-of-band зондов (fs/net/mem/proc/syscall) с настоящей
  Ed25519-подписью и реальным Kafka-transport — отдельный заход; пока репо
  держит стартовый скелет (config/transport/command), который доводится до
  out-of-band.

## Alternatives considered

- **In-guest агент**: отвергнут — ненадёжен в скомпрометированной среде.
- **Go/Python для коллектора**: возможны, но Rust даёт лучший профиль для
  per-host демона с доверием.

## Related

- [`0002-trust-boundary.md`](0002-trust-boundary.md).
- [`../COMPOSITION.md`](../COMPOSITION.md) — «Имена и обоснование».