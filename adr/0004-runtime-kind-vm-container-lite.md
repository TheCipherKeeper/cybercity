# ADR-0004: runtime_kind {vm, container, lite} + honeypot-назначение; движок — регистратор

## Status

Accepted

Supersedes:

- [`cybercity-engine/docs/adr/0003-hybrid-execution.md`](https://github.com/TheCipherKeeper/cybercity-engine/blob/main/docs/adr/0003-hybrid-execution.md)
  (ранее режимы `real` / `simulated` / `decoy`);
- [`cybercity-data/docs/adr/0004-service-decoy-mock.md`](https://github.com/TheCipherKeeper/cybercity-data/blob/main/docs/adr/0004-service-decoy-mock.md)
  (поле `Service.decoy`) — заменяется локальным
  [`cybercity-data/docs/adr/0019-service-honeypot-purpose.md`](https://github.com/TheCipherKeeper/cybercity-data/blob/main/docs/adr/0019-service-honeypot-purpose.md).

## Context

В ADR-0003 (engine) модель исполнения сервисов задавалась тремя режимами
`{real, simulated, decoy}`:

- `real` — внешняя VM/контейнер, наблюдаемая коллектором;
- `simulated` — «хост», который **эмулятор движка выдумывает** по метаданным
  (φ(v) + состояние); реального сокета/процесса нет, не снампится по TCP;
- `decoy` — одновременно и runtime-режим, и блок-наживки в `cybercity-data`, и
  «simulation-only mock».

На практике модель оказалась неудобна для оператора-человека: «simulated» не
существует как runnable-сущность (его нельзя `docker ps` / `qm list`, нельзя
сканировать реальным инструментом), а «decoy» перегружен тремя смыслами. При этом
в коде движка «simulated» так и осталось мёртвыми константами (`ServiceMode`,
определён, но нигде не хранится и не диспетчится; эмулятор не реализован) —
аспирация, не построенное.

В `cybercity-data` же «decoy» — это рабочий блок-наживка (`{kind, fingerprint,
os_hint, note}`) с cross-field-правилами (`decoy-criticality`, `decoy-write-real`),
уже фактически использующийся как honeypot-назначение (см. `organizations/isp`,
`isp-honeypot`).

## Decision

Две независимые оси вместо одной перегруженной:

### 1. `runtime_kind` — как сервис исполняется (deployment-time)

У каждого сервиса есть `runtime_kind ∈ {vm, container, lite}`:

- `vm` — полная виртуальная машина, настоящая ОС/ядро, real software (Windows,
  OT/SCADA, persistence). Reset — ZFS snapshot/clone.
- `container` — контейнер с настоящим ПО на shared-ядре; gVisor/Kata для
  adversarial-изоляции. Reset — restart pod.
- `lite` — **максимально лёгкий stub-контейнер**: реальный сокет + подделанный
  баннер/поведение, параметризуется дескриптором сервиса (`kind`, `ports`,
  `software`, `cve_id`, `honeypot`). Заменяет «simulated». Реализуется
  переиспользуемым образом `cc-lite` (TODO — см. «Out of scope»). Reset —
  restart pod.

`runtime_kind` — **deployment-time concern, не часть канонической city data.**
Назначается в `cybercity-manage` service-mapping manifest
(`service_id → {runtime_kind, template, honeypot?}`), не живёт в
`cybercity-data/config.yml` и не попадает в `engine.zip`. По умолчанию
(`runtime_kind` не указан) — `lite`: фон города должен быть дёшев, а дорогие
vm/container — только для целей сценария.

### 2. `honeypot` — назначение-наживка (свойство сервиса, в data)

`honeypot` — булев флаг + блок fingerprint (бывший `decoy`): сервис,
предназначенный bait'ить сканирования/атаки, собирать threat intel. Это
**назначение**, ортогональное `runtime_kind`: honeypot может быть реализован как
`lite` (обычный случай) или `vm` (реальный honeypot-VM). Cross-field-правила
`honeypot-criticality` (не `critical`) и `honeypot-write-real` (не пишет в
реальные сервисы) остаются как ограничения назначения.

Слово **«decoy» упраздняется** в пользу `honeypot` (назначение) и `lite`
(runtime-вид). «simulated» упраздняется в пользу `lite`.

### 3. Движок — регистратор, не симулятор

`cybercity-engine` **не вычисляет исходы взаимодействий** для сервисов. Все
runtime-цели (`vm`/`container`/`lite`) — runnable и наблюдаются
`cybercity-collector` out-of-band единообразно; исход (scan, compromise, state
change) приходит как подписанное событие от коллектора. Движок записывает
события, ведёт причинный граф, считает scoring — но не «отыгрывает» сервисы.

Следствие: класса «engine-synthesized service events» больше нет. Это
упрощает доверительную границу (ADR-0002): все service-события —
коллектор-наблюдаемые (доверенные); второго (best-effort из движка) класса
service-исходов нет. In-guest enrichment остаётся best-effort — но это про
данные *изнутри* гостя, не про исходы, выдуманные движком.

## Контрактное изменение (data → engine → ui)

Переименование `decoy` → `honeypot` меняет артефакт `engine.zip` /
`topology.json` / `attack-surface.json`:

| было | стало |
|---|---|
| `is_mock` | `is_honeypot` |
| `mock_services` | `honeypot_services` |
| `decoy_profile` | `honeypot_profile` |
| `is_decoy` (engine) | `is_honeypot` |
| `decoy_kind` (engine) | `honeypot_kind` |

`runtime_kind` в артефакты data **не добавляется** (deployment-time). Все три
потребителя (data-билдер, engine-loader, UI) правятся согласованно; статическая
копия `topology.json` в UI перегенерируется.

## Consequences

### Positive

- Мёртво-простая ментальная модель: каждая сущность топологии — runnable
  артефакт (`qm list` + `kubectl get pods` покажут весь город). Никаких
  «призрачных хостов».
- Real-инструменты (`nmap` и т.п.) работают по всему городу, включая `lite`
  (реальный сокет + подделанный баннер).
- Единая доверительная модель: всё наблюдается коллектором → scoring
  однороден, класс engine-synth service-событий исчезает (упрощение ADR-0002).
- Движок меньше: уходит подсистема эмулятора (`internal/simulator` per-kind
  хендлеры + правила успеха). Движок = consumer-loop + граф + scoring + API.
- `runtime_kind` и `honeypot` — две чистые оси вместо одной перегруженной.

### Negative

- Логика фейковых сервисов **релоцируется**, не исчезает: правила «как lite
  отвечает» переезжают из движка в образ `cc-lite` (TODO). Пока образ не
  построен, `lite`-цель — заявленный контракт, не реализованная runtime-цель.
- N запущенных штук вместо ~2: `lite`-контейнеры потребляют Multus-IP, pod'ы,
  покрытие коллектором. На home lab ~300 — терпимо; за ~1000 — тяжелее, чем
  «бесплатный фон» in-engine `simulated` (которого в коде всё равно не было).
- Коллектор должен крыть все runtime-виды (был — горстка real-ВМ).
- Сервис-id `decoy-printer-01` и блок `decoy:` в фикстурах/артефактах
  переименовываются → ripple по графу и UI-копии `topology.json`.

## Alternatives considered

- **Оставить `{real, simulated, decoy}` (ADR-0003).** Отвергнуто: «simulated»
  неудержимо в голове оператора и не снампится; «decoy» перегружен; в коде
  `simulated` — мёртвые константы без эмулятора.
- **`runtime_kind ∈ {vm, container, decoy}` (decoy = лёгкий контейнер).**
  Отвергнуто: «decoy» коннотирует наживку, а не механизм; коллизует с
  data-блоком-наживкой; пришлось бы переименовывать блок в `honeypot` и
  оставлять decoy-как-контейнер — двойной смысл слова.
- **In-engine `simulated` + real (гибрид ADR-0003), но достроить эмулятор.**
  Отвергнуто: плодит второй класс событий и подсистему эмулятора; для
  человеко-управляемого полигона uniform-observation проще.

## Out of scope (TODO, фиксируется здесь)

- Образ `cc-lite` (Dockerfile + параметризуемый бинарь: биндит порты, подделанный
  баннер, heartbeat коллектору).
- Python-код `cybercity-manage` (ports/adapters/application) + загрузка
  service-mapping manifest.
- Engine: загрузка `runtime_kind` из manifest + диспетч (сейчас движок и так не
  симулирует — регистратор уже по факту).
- Collector: выбор probe-set по `runtime_kind`.
- UI: виджеты/фильтры по `runtime_kind`/`honeypot`.

## Related

- [`ADR-0002`](0002-trust-boundary.md) — доверительная граница; упрощается
  (engine-synth service-событий нет).
- [`cybercity-engine/docs/adr/0003-hybrid-execution.md`](https://github.com/TheCipherKeeper/cybercity-engine/blob/main/docs/adr/0003-hybrid-execution.md)
  — Superseded.
- [`cybercity-data/docs/adr/0019-service-honeypot-purpose.md`](https://github.com/TheCipherKeeper/cybercity-data/blob/main/docs/adr/0019-service-honeypot-purpose.md)
  — локальное data-решение про `Service.honeypot`.
- [`cybercity-manage/docs/adr/0002-runtime-kind-manifest.md`](https://github.com/TheCipherKeeper/cybercity-manage/blob/main/docs/adr/0002-runtime-kind-manifest.md)
  — service-mapping manifest + `cc-lite`.