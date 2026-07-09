# ADR-0010: `cybercity-data` как broker-участник

## Status

Accepted

## Scope

`data` (затрагивает `CONVENTIONS@v1`: новый топик `city.build.completed`).

## Context

`cybercity-data` исторически — **чистый CLI-инструмент**: `cybercity-data build`
рендерит декларативную модель города в файловые артефакты (`engine.zip`,
`topology.json`, `attack-surface.json`, `schema.json`, `overlays`, артефакт
сценария). Потребители (`engine`, `ui`, `manage`) берут эти артефакты
**out-of-band** (файлы — MinIO/файловая шара), без брокера. В этой модели `data`
— не участник общения, а источник файлов.

Методология (`<methodology-repo>/docs/refs/COMMUNICATION.md`) определяет сервис
как **клиента брокера** (publish/consume топики); прямые service-to-service связи
в обход брокера запрещены. Файловые артефакты out-of-band не нарушают это правило
(они не «service-to-service вызов»), но оставляют `data` вне брокерной модели:
другие сервисы не получают события о готовности сборки и вынуждены поллить
файловую шару или тянуть артефакт вручную.

Профиль `data` прояснился:

- **Сборка — событие, а не только файл.** Готовый `engine.zip`/`overlays` — это
  момент, о котором `engine` и `manage` хотят узнать сразу (engine — чтобы
  reload топологии; manage — чтобы пересобрать образы из нового `overlays`).
  Поллинг файловой шары — задержка + лишняя связность.
- **`data` уже зрелый сервис** (модель, валидация, аллокатор, сборка артефактов,
  CI 95% coverage, mypy --strict). Шаг от CLI к broker-producer — добавление
  одного output port, а не переписывание.
- **Конверт уже есть** (`CONVENTIONS@v1`): `city.build.completed` естественно
  ложится в `event_type` с `source_type: system`/`producer: data`.

## Decision

`cybercity-data` становится **broker-участником** (producer): помимо файловых
артефактов, публикует событие готовности сборки в топик `city.build.completed`.

- **Топик:** `city.build.completed` (publish, `producer: data`,
  `source_type: system`). Payload: ссылки на артефакты (`engine.zip`,
  `overlays`, сценарий), `seed`, `build_id`, контрольная сумма.
- **Файловые артефакты остаются** (`engine.zip`/`topology.json`/`overlays`/
  сценарий) — потребляются `engine`/`ui`/`manage` out-of-band как раньше.
  Событие — **уведомление о готовности**, не замена артефакта.
- **`CONVENTIONS@v1` расширяется** новым `event_type` `city.build.completed`
  (backward-compatible: новое событие, не изменение существующего поля — без
  bump, остаёмся `@v1`). Пин `data` — `CONVENTIONS@v1`.
- **Code seams (Phase 2):** юзкейс build/check получает output port
  `EventPublisher` (в `ports/`), реализованный в `adapters/` Redpanda-
  публикатором envelope `CONVENTIONS@v1`. Существующий layered-код
  (Controller→Service→UseCase→Domain/Data) перестраивается в per-module
  `usecases/ports/domain/adapters` с `Protocol` ports + broker-adapter
  (`<methodology-repo>/docs/refs/MODULE.md`).
- **`data` НЕ становится consumer'ом** на этом этапе: он только публикует.
  Consumer-цикл (если понадобится — напр. реакции на infra-события) — отдельный
  ADR.

**Что НЕ меняет этот ADR:**

- **Файловые контракты** `data → engine/ui` (`engine.zip`, `topology.json`),
  `data → manage` (`overlays`), `data → engine` (сценарий) — остаются
  out-of-band. Брокерное событие — дополнение, не замена.
- **`data` как source of truth модели/сценариев/уязвимостей** —
  не затронуто; `engine` по-прежнему исполняет, не авторит.
- **Доверительная граница** — `data` живёт в control-плоскости (не в range);
  публикация в брокер из control-плоскости не нарушает изоляцию range.

## Consequences

### Positive

- `engine`/`manage` получают событие о готовности сборки без поллинга —
  реактивная reload/пересборка.
- `data` полноценно ложится в брокерную модель методологии (сервис = broker
  client), закрывая рассинхронизацию «CLI vs сервис».
- Backward-compatible (`@v1` без bump): существующие consumer'ы не ломаются.

### Negative

- **Продуктовое изменение сути `data`** (CLI → producer): меняет цикл и
  операционную модель; требует, чтобы `data` имел адрес брокера и пин
  (`BROKER_ADDR`, `CONVENTIONS_PIN=v1`) даже для разовых сборок (или
  publisher-fallback в offline — TBD в спеке).
- **Новый adapter** (Redpanda-публикатор) + его тесты — стоимость Phase 2.
- **Идемпотентность** по `event_id`: повторная сборка того же `build_id` не
  должна публиковать дубль (consumer дедуплицирует, но producer обязан быть
  детерминированным по `--seed`).

## Alternatives considered

- **Оставить `data` чистым CLI (файлы out-of-band, без брокера).** Отвергнуто —
  поллинг файловой шары — задержка и лишняя связность; `data` остаётся вне
  брокерной модели методологии. См. Context.
- **Полностью заменить файловые артефакты брокерными сообщениями** (тащить
  `engine.zip` через брокер). Отвергнуто — крупные бинарные артефакты не
  принадлежат event bus; событие — уведомление, артефакт — файл. Стандартная
  pattern (claim-check).
- **Сделать `data` consumer'ом тоже (реагировать на infra-события).** Отвергнуто
  на этом этапе — нет текущей потребности; потребительский цикл — отдельным ADR,
  когда появится.

## Related

- [ADR-0001](0001-repo-composition.md) — семь репозиториев; `data` — декларативная
  модель + авторинг. Основной тезис в силе; этот ADR добавляет broker-роль.
- [ADR-0003](0003-collector-rust-out-of-band.md) — брокер Redpanda.
- [ADR-0006](0006-vulnerability-declarative-overlay-realism.md) — `overlays`-
  артефакт (контракт `data → manage`, остаётся out-of-band).
- [`../COMPOSITION.md`](../COMPOSITION.md) — строка `data` (publish
  `city.build.completed`), «Потоки данных и контракты» (пункт 4).
- [`../CONVENTIONS.md`](../CONVENTIONS.md) — `CONVENTIONS@v1`, envelope, топик
  `city.build.completed`.
- `<methodology-repo>/docs/refs/COMMUNICATION.md` — сервис как broker-клиент.