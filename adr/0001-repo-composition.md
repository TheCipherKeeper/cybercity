# ADR-0001: Семь репозиториев

## Status

Accepted — выбор стека `cybercity-manage` (Python) superseded в
[ADR-0009](0009-manage-implementation-language-go.md): manage реализуется на
Go (`bpg/proxmox-go-sdk`, `hashicorp/terraform-exec`/Pulumi Go SDK). Основной
тезис (семь репозиториев, состав) не изменён; строки про manage ниже обновлены
под новое решение.

## Scope

Сквозное.

## Context

CyberCity — это слоёная система, где разные слои живут на разных стеках
(Python для данных, Go для runtime и контрольной плоскости, Rust для
коллектора и lite-целей, TS для UI), имеют разные циклы зрелости и разные CI/линтер-цепочки. Размещать всё это
в одном дереве мешало бы независимой эволюции слоёв и снижало строгость: в
единой большой репе слои легко начинают дрейфовать друг от друга.

## Decision

Разделить проект на 7 репозиториев — по одному слайсу системы в каждом:

- `cybercity` — обложка/индекс + системные документы (без кода);
- `cybercity-data` — декларативная модель + авторинг сценариев (Python);
- `cybercity-engine` — event-driven runtime (Go);
- `cybercity-manage` — контрольная плоскость (Go поверх Proxmox/IaC);
- `cybercity-collector` — out-of-band per-host коллектор (Rust);
- `cybercity-clite` — параметризуемый stub-образ `clite` для `runtime_kind: lite` (Rust);
- `cybercity-ui` — web-фронтенд (TS).

Системное видение, архитектура, конвенции и сквозные ADR живут в хабе
`cybercity`; каждый репозиторий — только то, как реализовано в нём.

### Почему `cybercity-clite` отдельным репо

`clite` — единственный артефакт системы, который живёт **внутри range-сегмента**
(ненадёжная плоскость): он и есть наблюдаемая `lite`-цель. Положить его в
`cybercity-manage` (Go) — не тот стек; в `cybercity-collector` (Rust, но
mgmt/out-of-band) — смешивает range- и mgmt-артефакты в одном репо и размывает
доверительную границу (ADR-0002). Отдельный репо на Rust держит границу чистой,
даёт образу свой релизный цикл (Docker-образ, деплоящийся per-lite-target по
всему городу) и совпадает по стеку с коллектором — общие скиллы, возможны общие
крейты (event envelope, подпись).

Имя `cybercity-clite` (не `cybercity-lite`), чтобы не читалось как «облегчённая
cybercity»; образ/бинарь — `clite` (от «container lite»). См. также ADR-0004,
где `lite` — дефолтный `runtime_kind`.

## Consequences

### Positive

- Каждый слой эволюционирует на своём стеке и своём ритме.
- Чёткие границы ответственности = меньше дрейфа между слоями; range-артефакт
  не смешан с mgmt-репо.
- Хаб как единый источник правды о составе и контрактах.

### Negative

- Контракты между репозиториями надо поддерживать явно (схемы, event envelope,
  дескриптор сервиса → поведение `clite`).
- Перекрёстные ссылки — через GitHub URL, не через файловые пути.

## Alternatives considered

- **Монорепа**: единое дерево со всеми слоями — отвергнуто (стеки и циклы
  разные, единое дерево мешает независимой эволюции слоёв).
- **Монорепа + git submodules**: гибрид — лишняя сложность git, не оправдано.
- **`clite` внутри `cybercity-collector`**: отвергнуто — range-артефакт в
  mgmt-репо размывает доверительную границу (ADR-0002).

## Related

- [`../COMPOSITION.md`](../COMPOSITION.md) — состав и контракты.
- [`../CONVENTIONS.md`](../CONVENTIONS.md) — скелет репозитория, иерархия доков.
- [`0002-trust-boundary.md`](0002-trust-boundary.md) — почему `clite` не может
  жить в mgmt-репо.
- [`0004-runtime-kind-vm-container-lite.md`](0004-runtime-kind-vm-container-lite.md)
  — `runtime_kind: lite`, который реализует `clite`.
- [`0009-manage-implementation-language-go.md`](0009-manage-implementation-language-go.md)
  — Go как язык реализации manage; supersede стек-части этого ADR.