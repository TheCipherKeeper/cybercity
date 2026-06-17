# CyberCity

[![Part of CyberCity](https://img.shields.io/badge/CyberCity-composition-blueviolet)](#)
[![License: MIT](https://img.shields.io/badge/code-MIT-green)](LICENSE)
[![Docs: CC BY 4.0](https://img.shields.io/badge/docs-CC%20BY%204.0-lightgrey)](LICENSE-DOCS)

**CyberCity** — модульный кибер-полигон. Симулирует целый город
(больницы, электросеть, транспорт, банки, суды) для учений
red / blue team.

Этот репозиторий — **обложка** проекта: здесь нет кода, но лежат
**системные** документы — видение, архитектура, конвенции и сквозные ADR.
Реализация каждого слоя — в отдельном репозитории (`cybercity-*`).

> Исходная точка: [`VISION.md`](VISION.md) — философия и дорожная карта.
> Этот README — короткая выжимка; канон композиции — в
> [`COMPOSITION.md`](COMPOSITION.md).

## Идея

Город — это **данные**. Симуляция, визуализация, отчёты, учения —
проекции одного декларативного описания. Если данные хорошие,
остальное получается само. Если плохие — никакой код не спасёт.

Учения встраиваются в живую жизнь города. Атаковал светофоры — на
карте реально меняется картинка, в отчёте это отражено. Не MMORPG,
не сценарий-в-вакууме.

## Документация в этом репозитории

| Документ | Назначение |
|----------|-----------|
| [`COMPOSITION.md`](COMPOSITION.md) | Канон состава: репозитории, контракты, доверительная граница, история переименований. |
| [`VISION.md`](VISION.md) | Философия, принципы, аудитории, критерии успеха, non-goals. |
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | Системная архитектура: контекст, ответственности, два графа, слои развёртывания. |
| [`CONVENTIONS.md`](CONVENTIONS.md) | Кросс-репо конвенции: иерархия документов, язык, скелет репо, ADR/README-формат, event envelope. |
| [`adr/`](adr/) | Сквозные архитектурные решения (почему 6 репо, доверительная граница, Rust-коллектор). |

## Композиция

Каноническая карта слоёв, контрактов и доверительной границы —
[`COMPOSITION.md`](COMPOSITION.md). Ниже — короткая выжимка.

```
                    ┌──────────────────────────┐
                    │      cybercity (cover)   │ ← вы здесь
                    └─────────────┬────────────┘
                                  │ системные документы
        ┌────────────┬────────────┼────────────┬────────────┐
        ▼            ▼            ▼            ▼            ▼
   cybercity-    cybercity-    cybercity-   cybercity-   cybercity-
     data         engine          ui       collector     manage
 (model+сцен.) (Go runtime)  (2D-карта) (out-of-band) (control plane)
```

Каждый репозиторий — один слайс системы. Одна и та же сетевая модель,
разные слои. Никакого SaaS, никакой телеметрии к автору, всё крутится
на твоём Proxmox / K8s.

| | Репозиторий | Что делает |
|---|---|---|
| 🎯 | **[cybercity](https://github.com/TheCipherKeeper/cybercity)** | Обложка и системные документы (этот репо); канон композиции — [`COMPOSITION.md`](COMPOSITION.md) |
| 🗺️ | [cybercity-data](https://github.com/TheCipherKeeper/cybercity-data) | Декларативная модель города (source of truth) + авторинг сценариев (Python) |
| ⚙️ | [cybercity-engine](https://github.com/TheCipherKeeper/cybercity-engine) | Go runtime: событийное ядро, причинный граф, replay, эмуляция трафика, scoring |
| 🏗️ | [cybercity-manage](https://github.com/TheCipherKeeper/cybercity-manage) | Контрольная плоскость: provisioning, reset/rollback, изоляция, квоты (Python поверх Proxmox/Terraform) |
| 📡 | [cybercity-collector](https://github.com/TheCipherKeeper/cybercity-collector) | Внешний out-of-band per-host коллектор (Rust): подписанные события в engine по Kafka |
| 🖥️ | [cybercity-ui](https://github.com/TheCipherKeeper/cybercity-ui) | 2D-карта топологии, таймлайн событий, дашборды red/blue, отчёты |

Полная карта и общий стиль: [`TheCipherKeeper`](https://github.com/TheCipherKeeper/TheCipherKeeper) (профиль) и [thecipherkeeper.github.io](https://thecipherkeeper.github.io) (CV, развёрнутые описания).

## Как читать

1. **Композиция** — [`COMPOSITION.md`](COMPOSITION.md): репозитории, контракты, доверительная граница, история переименований.
2. **Философия** — [`VISION.md`](VISION.md): что строим, зачем, в каком порядке.
3. **Архитектура** — [`ARCHITECTURE.md`](ARCHITECTURE.md): системный контекст, два графа, слои.
4. **Конвенции** — [`CONVENTIONS.md`](CONVENTIONS.md): иерархия доков, язык, скелет репо, форматы.
5. **Каноническая модель города** — `organizations/*/config.yml` в [`cybercity-data`](https://github.com/TheCipherKeeper/cybercity-data) и его человекочитаемая проекция `build/network.md`.
6. **Runtime** — [`cybercity-engine`](https://github.com/TheCipherKeeper/cybercity-engine): событийное ядро, причинный граф, replay, эмуляция трафика.
7. **Управление и сбор** — [`cybercity-manage`](https://github.com/TheCipherKeeper/cybercity-manage) (контрольная плоскость) + [`cybercity-collector`](https://github.com/TheCipherKeeper/cybercity-collector) (out-of-band коллектор).
8. **Визуализация** — [`cybercity-ui`](https://github.com/TheCipherKeeper/cybercity-ui): 2D-карта, таймлайн, дашборды.

## Принципы

- **События — единственный источник истины.** Состояние = проекция потока.
- **Сеть декларативна.** Один YAML описывает всё. K8s — его проекция.
- **Безопасность по умолчанию.** Сегменты изолированы, каналы — явные. OT не светится наружу.
- **LLM — помощник, не хозяин.** LLM пишет YAML/код, валидаторы и тесты решают.
- **Воспроизводимость.** Один вход → один выход. Детерминированный режим, фиксированный порядок ключей.

## Как я работаю

(Выжимка из [`CONVENTIONS.md`](CONVENTIONS.md) § «LLM — помощник, не хозяин» и репозиторных `AGENTS.md`.)

- По одной сущности за итерацию. Не «30 организаций сразу».
- Валидатор и тесты — единственный контракт правды. Каждое AI-изменение через `lint`/`test`.
- LLM пишет YAML и код; валидаторы, линтеры и человек решают, что принято.
- Коммиты и пуши — вручную. AI не пушит без явного одобрения.

## Статус

В активной разработке. Модель города: 46 организаций / 263 сервиса / 464 линка; сценарии учений авторятся в `cybercity-data` (формат в разработке).
Дорожная карта — в [`VISION.md`](VISION.md) и [`ARCHITECTURE.md`](ARCHITECTURE.md) (§ «Дорожная карта»).

## Лицензия

- Код: [MIT](LICENSE)
- Документация: [CC BY 4.0](LICENSE-DOCS)