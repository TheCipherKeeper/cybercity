# ADR-0009: Go как язык реализации `cybercity-manage`

## Status

Accepted

## Scope

`manage`.

## Context

ADR-0001 зафиксировал стек реализации `cybercity-manage` как **Python +
`proxmoxer` + `python-terraform` / CDKTF**, сознательно клонируя
инструментарий зрелого `cybercity-data` (ruff / mypy --strict / pytest-cov 95%
/ hypothesis). На момент принятия ADR-0001 это был осмысленный выбор «тонкого
слоя над реальным IaC».

С тех пор профиль manage прояснился, и ряд свойств делает Python неочевидным:

- **Long-running control plane.** manage — не разовый `apply`, а постоянно
  живущая плоскость, оркеструющая гипервизор/фабрику в ответ на команды и
  события. Профиль — сервис, а не скрипт.
- **Единственный инфра-мутатор на доверительной границе.** Строгая
  статическая типизация и предсказуемая память здесь к лицу.
- **Параллельный provisioning.** Reset/провижнинг идёт по множеству хостов
  одновременно — нужна дешёвая конкурентность.
- **Модель деплоя.** manage живёт в mgmt-сегменте рядом с
  `cybercity-collector` (Rust, single static binary). Совпадение бинарной
  модели деплоя упрощает эксплуатацию доверенной плоскости.
- **Кода ещё нет.** Момент пересмотреть стек-решение без стоимости миграции.

Экосистема Go под задачу не хуже, а местами роднее, чем Python:

- **`hashicorp/terraform-exec`** — канонический Go-SDK для управления
  Terraform CLI (зрелее и идиоматичнее, чем `python-terraform`).
- **Pulumi Go SDK** — нативный SDK для декларативных частей.
- **`bpg/proxmox-go-sdk`** — зрелый клиент к Proxmox API (REST).

## Decision

Язык реализации `cybercity-manage` — **Go (≥ 1.23)**; модуль
`github.com/TheCipherKeeper/cybercity-manage`.

- **Proxmox API** — через `bpg/proxmox-go-sdk` (REST): создание/удаление/
  старт/стоп гостей, snapshot/clone.
- **Terraform/Pulumi как библиотека** — `hashicorp/terraform-exec` (применение
  сгенерированных конфигов) и/или **Pulumi Go SDK** для декларативных частей,
  где это уместно; CDKTF-on-Go — опционально.
- **Single static binary.** manage собирается в один бинарник без runtime-
  зависимостей — модель деплоя совпадает с Rust-коллектором.
- **Конкурентность** — горутины для параллельного provisioning/reset по
  множеству хостов; `context.Context` — сквозной для отмены/таймаутов.
- **Ports & adapters (onion) сохраняется.** Порты — Go-интерфейсы в
  `internal/ports`, адаптеры — реализации в `internal/adapters`; `domain` не
  знает про Proxmox/Terraform/ZFS (см. `cybercity-manage/docs/ARCHITECTURE.md`).

**Что НЕ меняет этот ADR:**

- **Основной тезис ADR-0001** (семь репозиториев; manage — контрольная
  плоскость над реальным IaC, не переописание provisioning) остаётся в силе.
  Superseded только выбор Python + `proxmoxer` + `python-terraform`/CDKTF как
  стека *реализации* manage.
- **ADR-0002** (service-mapping manifest, `runtime_kind {vm, container, lite}`,
  образ `clite`) не затронут: manifest по-прежнему YAML, ownership и
  runtime-модель те же.
- **Доверительная граница** (manage + collector + Kafka в mgmt, без маршрута
  из range) — языково-ортогональна.

## Consequences

### Positive

- Single static binary, низкий footprint, модель деплоя = Rust-коллектор.
- Нативная конкурентность (горутины + `context`) для параллельного
  provisioning/reset по множеству хостов.
- Строгая статическая типизация — к лицу единственному инфра-мутатору.
- Канонические SDK (`terraform-exec`, Pulumi Go SDK) — зрелее/идиоматичнее
  Python-аналогов.

### Negative

- **Разрыв сквозной Python-конвенции с `cybercity-data`** (и, возможно,
  `cybercity-engine`): второй тулчейн для команды.
- **Потеря готового Python-инструментария** (ruff / mypy --strict / pytest-cov
  95% / hypothesis) — нужно выстраивать Go-аналог (golangci-lint, `go test
  -coverprofile`, `pgregory.net/rapid`).
- **`bpg/proxmox-go-sdk`** требует валидации покрытия API под нашими
  сценариями (snapshot/clone, firewall, VLAN) — `proxmoxer` де-факто зрелее;
  риск обнаружить дыры в покрытии SDK на интеграции.
- Код пока отсутствует: документ фиксирует цель, к которой идём.

## Alternatives considered

- **Остаться на Python (ADR-0001 as-is).** Отвергнуто — управление решило
  сменить стек под long-running control-plane профиль; кода нет, стоимость
  смены нулевая. См. Context.
- **Go + свой provisioning с нуля.** Отвергнуто — то же, что и альтернатива
  ADR-0001: дублирование Proxmox/Terraform, мимо экосистемы, дорого и хрупко.
- **Rust (как `cybercity-collector`).** Отвергнуто — коллектор это out-of-band
  агент с жёсткими требованиями к памяти/подписям; control plane не имеет тех
  же ограничений, а Go даёт нужное с меньшей стоимостью разработки и единой
  бинарной моделью деплоя. См.
  [ADR-0003](0003-collector-rust-out-of-band.md).

## Related

- [ADR-0001](0001-repo-composition.md) — семь репозиториев; *стек-часть про
  manage superseded этим ADR*, основной тезис в силе.
- [ADR-0002](0002-trust-boundary.md) — доверительная граница; языково-
  ортогональна.
- [ADR-0003](0003-collector-rust-out-of-band.md) — почему коллектор на Rust
  (контраст к выбору Go для manage).
- [`../COMPOSITION.md`](../COMPOSITION.md) — состав; строка manage обновлена
  на Go.
- `cybercity-manage/docs/ARCHITECTURE.md`, `…/DEVELOPMENT.md` — целевой Go-стек
  и инструментарий контрольной плоскости.