# ADR-0002: Доверенная vs best-effort плоскость

## Status

Accepted

## Context

Scoring и replay требуют потока событий, которому можно доверять. Но гости
в range-сегменте по определению ненадёжны: их могли скомпрометировать, в них
может быть in-guest агент, который врёт. Если считать scoring на потоке из
гостей — атакующий подделывает свой результат.

## Decision

Две плоскости:

- **Trusted (доверенная):** `cybercity-manage` + `cybercity-collector` +
  Kafka-брокер, живут в mgmt-сегменте, **без маршрута из range**. На их потоке
  считается scoring.
- **Best-effort:** всё внутри гостей (включая опциональный in-guest
  enrichment) — **никогда** не источник для scoring.

`cybercity-collector` наблюдает гостей **снаружи** (out-of-band, read-only),
подписывает события Ed25519; Kafka — mTLS + ACL; гости до брокера не
достукиваются структурно. Действие над гостем (reset/изоляция) — через
`cybercity-manage`/фабрику, не через in-guest агента.

## Consequences

### Positive

- Scoring неотличим от правды: атакующий не может подделать поток.
- Replay детерминирован на доверенном потоке.

### Negative

- Требует отдельной mgmt-сегментации и подписанной телеметрии — операционная
  сложность.
- In-guest данные (если нужны) — только best-effort, отдельным каналом.

## Alternatives considered

- **Доверять in-guest потоку**: отвергнуто — компрометация гостя ломает scoring.
- **Отказаться от real-гостей, всё simulated**: теряется hands-on ценность.

## Related

- [`../COMPOSITION.md`](../COMPOSITION.md) — «Доверительная граница».
- [`0003-collector-rust-out-of-band.md`](0003-collector-rust-out-of-band.md).