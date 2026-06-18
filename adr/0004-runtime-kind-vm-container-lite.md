# ADR-0004: runtime_kind {vm, container, lite}; движок — регистратор

## Status

Accepted

## Scope

Сквозное.

## Context

Нужна модель исполнения сервисов, которая: (1) даёт реальный hands-on —
`nmap` по городу что-то видит на сокете; (2) дёшево для массового фона города;
(3) не плодит второй класс событий, отдельный от наблюдаемых коллектором
(доверительная граница ADR-0002).

## Decision

### 1. `runtime_kind` — как сервис исполняется (deployment-time)

У каждого сервиса `runtime_kind ∈ {vm, container, lite}`:

- `vm` — полная виртуальная машина, настоящая ОС/ядро, real software (Windows,
  OT/SCADA, persistence). Reset — ZFS snapshot/clone.
- `container` — контейнер с настоящим ПО на shared-ядре; gVisor/Kata для
  adversarial-изоляции. Reset — restart pod.
- `lite` — **максимально лёгкий stub-контейнер**: реальный сокет + поддельный
  баннер/поведение, параметризуется дескриптором сервиса (`kind`, `ports`,
  `software`, `cve_id`). Реализуется образом `cc-lite` (репо `cybercity-clite`,
  см. ADR-0001). Дешёвый runnable-фон города. Reset — restart pod.

`runtime_kind` — **deployment-time concern, не часть канонической city data.**
Назначается в `cybercity-manage` service-mapping manifest
(`service_id → {runtime_kind, template}`), не живёт в
`cybercity-data/config.yml` и не попадает в `engine.zip`. По умолчанию
(`runtime_kind` не указан) — `lite`: фон города должен быть дёшев, а дорогие
vm/container — только для целей сценария.

### 2. Движок — регистратор, не симулятор

`cybercity-engine` **не вычисляет исходы взаимодействий** для сервисов. Все
runtime-цели (`vm`/`container`/`lite`) — runnable и наблюдаются
`cybercity-collector` out-of-band единообразно; исход (scan, compromise, state
change) приходит как подписанное событие от коллектора. Движок записывает
события, ведёт причинный граф, считает scoring — но не «отыгрывает» сервисы.

Следствие: класса «engine-synthesized service events» нет. Это упрощает
доверительную границу (ADR-0002): все service-события — коллектор-наблюдаемые
(доверенные); второго (best-effort из движка) класса service-исходов нет.
In-guest enrichment остаётся best-effort — но это про данные *изнутри* гостя,
не про исходы, выдуманные движком.

## Consequences

### Positive

- Мёртво-простая ментальная модель: каждая сущность топологии — runnable
  артефакт (`qm list` + `kubectl get pods` покажут весь город). Никаких
  «призрачных хостов».
- Real-инструменты (`nmap` и т.п.) работают по всему городу, включая `lite`
  (реальный сокет + поддельный баннер).
- Единая доверительная модель: всё наблюдается коллектором → scoring
  однороден, класса engine-synth service-событий нет (упрощение ADR-0002).
- Движок меньше: нет подсистемы эмулятора (per-kind хендлеры + правила успеха).
  Движок = consumer-loop + граф + scoring + API.

### Negative

- Логика фейковых сервисов **живёт в образе, не в движке**: правила «как lite
  отвечает» реализованы в образе `cc-lite` (репо `cybercity-clite`), а не в
  движке. Пока образ не построен, `lite`-цель — заявленный контракт, не
  реализованная runtime-цель.
- N запущенных штук вместо ~2: `lite`-контейнеры потребляют Multus-IP, pod'ы,
  покрытие коллектором. На home lab ~300 — терпимо; за ~1000 — тяжелее, чем
  гипотетический «бесплатный фон» in-engine (но такой фон в системе и не
  заложен — всё runnable).
- Коллектор должен крыть все runtime-виды, а не только горстку real-ВМ.

## Alternatives considered

- **`{real, simulated, decoy}`**: отвергнуто. `simulated` — «хост», который
  эмулятор движка выдумывает по метаданным (φ(v) + состояние), реального сокета
  нет, не снампится по TCP: не существует как runnable-сущность (нельзя
  `docker ps` / `qm list`, нельзя сканировать реальным инструментом) —
  аспирация, а не построенное. `decoy` перегружен тремя смыслами (runtime-режим,
  блок-наживка в data, «simulation-only mock»).
- **In-engine `simulated` + real, достроить эмулятор**: отвергнуто — плодит
  второй класс событий и подсистему эмулятора; для человеко-управляемого
  полигона uniform-observation проще.
- **`runtime_kind ∈ {vm, container, decoy}` (decoy = лёгкий контейнер)**:
  отвергнуто — «decoy» коннотирует наживку, а не механизм; пришлось бы вводить
  отдельный блок-наживку с другим именем — двойной смысл слова.
- **Ось `honeypot` (назначение-наживка) как отдельная ортогональная ось над
  `runtime_kind`**: отложено для MVP. Механизм «дешёвый фейковый сервис»
  полностью покрывается `lite`; назначение-наживка (bait-сервис для threat intel
  / blue-team detection) несёт контрактную стоимость (поле в data, артефакты,
  cross-field-правила, UI-фильтры) и оправдана только при blue-team/IR-сценариях,
  которых пока нет. Вернуться к ней можно отдельным ADR.

## Related

- [`0001-repo-composition.md`](0001-repo-composition.md) — репо `cybercity-clite`,
  где строится образ `cc-lite`.
- [`0002-trust-boundary.md`](0002-trust-boundary.md) — доверительная граница;
  упрощается (engine-synth service-событий нет).
- [`0003-collector-rust-out-of-band.md`](0003-collector-rust-out-of-band.md) —
  коллектор наблюдает все runtime-виды out-of-band.