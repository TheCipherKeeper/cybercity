# CyberCity

[![Part of CyberCity](https://img.shields.io/badge/CyberCity-composition-blueviolet)](#)
[![License: MIT](https://img.shields.io/badge/code-MIT-green)](LICENSE)
[![Docs: CC BY 4.0](https://img.shields.io/badge/docs-CC%20BY%204.0-lightgrey)](LICENSE-DOCS)

**CyberCity** — модульный кибер-полигон. Симулирует целый город
(больницы, электросеть, транспорт, банки, суды) для учений
red / blue team.

Этот репозиторий — **обложка** проекта: здесь нет кода, только системные
документы. Реализация каждого слоя — в отдельном репозитории (`cybercity-*`).
Этот README — роутер: что где читать. Канон состава — в
[`COMPOSITION.md`](COMPOSITION.md), философия — в [`VISION.md`](VISION.md).

## Идея (в двух словах)

Город — это **данные**. Симуляция, визуализация, отчёты, учения — проекциии
одного декларативного описания. Учения встраиваются в живую жизнь города:
атаковал светофоры — на карте реально меняется картинка. Не MMORPG, не
сценарий-в-вакууме. Полное обоснование и принципы — в [`VISION.md`](VISION.md).

## Документация в этом репозитории

| Документ | Ответ на вопрос | Что внутри |
|----------|-----------------|------------|
| [`VISION.md`](VISION.md) | Зачем? | Философия, принципы, аудитории, non-goals, критерии успеха. |
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | Как соединено? | Системный контекст, два графа, hybrid execution, сетевая топология и сегментация, слои развёртывания, roadmap. |
| [`DATA_FLOW.md`](DATA_FLOW.md) | Как течёт во времени? | Жизненный цикл события и сценария, end-to-end потоки, replay/scoring. |
| [`COMPOSITION.md`](COMPOSITION.md) | Что есть и какие контракты? | Репозитории, потоки данных, доверительная граница, ownership, имена, статус реализации. |
| [`CONVENTIONS.md`](CONVENTIONS.md) | Как работаем? | Иерархия документов, язык, скелет репо, ADR/README-формат, event envelope, лицензии. |
| [`adr/`](adr/) | Почему так? | Все архитектурные решения проекта (сквозные и локальные). |

## Навигация по слоям

```
                    ┌──────────────────────────┐
                    │      cybercity (cover)   │ ← вы здесь
                    └─────────────┬────────────┘
                                  │ системные документы
        ┌────────────┬────────────┼────────────┬────────────┐
        ▼            ▼            ▼            ▼            ▼            ▼
   cybercity-    cybercity-    cybercity-   cybercity-   cybercity-   cybercity-
     data         engine          ui       collector     manage       clite
 (model+сцен.) (Go runtime)  (2D-карта) (out-of-band) (control plane) (lite-цель)
```

Авторитетная таблица репозиториев (слой / язык / назначение) — в
[`COMPOSITION.md`](COMPOSITION.md). Ниже — короткая карта-визуал для ориентирования.

| | Репозиторий | Роль |
|---|---|---|
| 🎯 | **[cybercity](https://github.com/TheCipherKeeper/cybercity)** | Обложка и системные документы (этот репо) |
| 🗺️ | [cybercity-data](https://github.com/TheCipherKeeper/cybercity-data) | Декларативная модель города + авторинг сценариев (Python) |
| ⚙️ | [cybercity-engine](https://github.com/TheCipherKeeper/cybercity-engine) | Go runtime: событийное ядро, причинный граф, replay, scoring |
| 🏗️ | [cybercity-manage](https://github.com/TheCipherKeeper/cybercity-manage) | Контрольная плоскость: provisioning, reset, изоляция, квоты (Python) |
| 📡 | [cybercity-collector](https://github.com/TheCipherKeeper/cybercity-collector) | Внешний out-of-band per-host коллектор (Rust) |
| 🧩 | [cybercity-clite](https://github.com/TheCipherKeeper/cybercity-clite) | Stub-образ `clite` для `runtime_kind: lite` (Rust) |
| 🖥️ | [cybercity-ui](https://github.com/TheCipherKeeper/cybercity-ui) | 2D-карта, таймлайн, дашборды red/blue (TS) |

Профиль автора: [`TheCipherKeeper`](https://github.com/TheCipherKeeper/TheCipherKeeper) · [thecipherkeeper.github.io](https://thecipherkeeper.github.io).

## Как читать

1. **Зачем** — [`VISION.md`](VISION.md): что строим, для кого, какие принципы.
2. **Что и контракты** — [`COMPOSITION.md`](COMPOSITION.md): репозитории, потоки, доверительная граница, ownership.
3. **Как соединено** — [`ARCHITECTURE.md`](ARCHITECTURE.md): системный контекст, два графа, сетевая топология, слои, roadmap.
4. **Как течёт во времени** — [`DATA_FLOW.md`](DATA_FLOW.md): жизненный цикл события и сценария, end-to-end потоки, replay/scoring.
5. **Как работаем** — [`CONVENTIONS.md`](CONVENTIONS.md): иерархия доков, язык, скелет репо, форматы.
6. **Почему так** — [`adr/`](adr/): все архитектурные решения (сквозные и локальные).

Текущий статус и дорожная карта — в [`ARCHITECTURE.md`](ARCHITECTURE.md) (§ «Дорожная карта»).
Принципы и подход к работе с LLM — в [`VISION.md`](VISION.md) и
[`CONVENTIONS.md`](CONVENTIONS.md) (§ «LLM — помощник, не хозяин»).

## Лицензия

Код — [MIT](LICENSE), документация — [CC BY 4.0](LICENSE-DOCS). Полное правило о
лицензиях в каждом репозитории — в [`CONVENTIONS.md`](CONVENTIONS.md) (§ «Лицензии»).