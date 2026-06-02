# CyberCity

[![Part of CyberCity](https://img.shields.io/badge/CyberCity-composition-blueviolet)](#)
[![License: MIT](https://img.shields.io/badge/code-MIT-green)](LICENSE)
[![Docs: CC BY 4.0](https://img.shields.io/badge/docs-CC%20BY%204.0-lightgrey)](LICENSE-DOCS)

**CyberCity** — модульный кибер-полигон. Симулирует целый город
(больницы, электросеть, транспорт, банки, суды) для учений
red / blue team.

Этот репозиторий — **обложка** проекта. Здесь нет кода, нет данных,
нет документации. Только индекс: что строим, зачем, где какой слой
лежит, в каком порядке читать.

> Исходная точка: [`master.md`](https://github.com/TheCipherKeeper/cybercity-engine/blob/main/master.md)
> — философия и дорожная карта. Этот README — короткая выжимка;
> полная версия — в engine-репо.

## Идея

Город — это **данные**. Симуляция, визуализация, отчёты, учения —
проекции одного декларативного описания. Если данные хорошие,
остальное получается само. Если плохие — никакой код не спасёт.

Учения встраиваются в живую жизнь города. Атаковал светофоры — на
карте реально меняется картинка, в отчёте это отражено. Не MMORPG,
не сценарий-в-вакууме.

## Композиция

```
                    ┌──────────────────────────┐
                    │      cybercity (cover)   │ ← вы здесь
                    └─────────────┬────────────┘
                                  │ индекс
        ┌────────────┬────────────┼────────────┬────────────┐
        ▼            ▼            ▼            ▼            ▼
   cybercity-    cybercity-    cybercity-   cybercity-   cybercity-
     data         engine          ui        agents     blueprints
  (network.yml)  (Go runtime)   (2D map)  (log/SIEM)  (Proxmox VMs)
```

Каждый репозиторий — один слайс системы. Одна и та же сетевая модель,
разные слои. Никакого SaaS, никакой телеметрии к автору, всё крутится
на твоём Proxmox / K8s.

| | Репозиторий | Что делает |
|---|---|---|
| 🎯 | **[cybercity](https://github.com/TheCipherKeeper/cybercity)** | Обложка и индекс проекта (этот репо) |
| ⚙️ | [cybercity-engine](https://github.com/TheCipherKeeper/cybercity-engine) | Go-каркас: событийное ядро, валидатор сети, рендер K8s-манифестов |
| 🗺️ | [cybercity-data](https://github.com/TheCipherKeeper/cybercity-data) | 30 организаций, 95 сервисов, decoy-хосты в `network.yml` |
| 🖥️ | [cybercity-ui](https://github.com/TheCipherKeeper/cybercity-ui) | 2D-карта города, таймлайн событий, дашборды red/blue |
| 📡 | [cybercity-agents](https://github.com/TheCipherKeeper/cybercity-agents) | Сборщики логов / SIEM-агенты внутри сегментов |
| 🏗️ | [cybercity-blueprints](https://github.com/TheCipherKeeper/cybercity-blueprints) | Provisioning Proxmox-VM узлов (Ansible / Terraform) |

Полная карта и общий стиль: [`TheCipherKeeper`](https://github.com/TheCipherKeeper/TheCipherKeeper) (профиль) и [thecipherkeeper.github.io](https://thecipherkeeper.github.io) (CV, развёрнутые описания).

## Как читать

1. **Философия** — [`master.md`](https://github.com/TheCipherKeeper/cybercity-engine/blob/main/master.md): что строим, зачем, в каком порядке.
2. **Каноническая модель города** — [`cybercity-data/network.yml`](https://github.com/TheCipherKeeper/cybercity-data/blob/main/network.yml) и его человекочитаемая проекция [`network.md`](https://github.com/TheCipherKeeper/cybercity-data/blob/main/network.md).
3. **Runtime** — [`cybercity-engine`](https://github.com/TheCipherKeeper/cybercity-engine): Go-валидатор, событийное ядро, рендер K8s-манифестов.
4. **Визуализация** — [`cybercity-ui`](https://github.com/TheCipherKeeper/cybercity-ui): 2D-карта, таймлайн, дашборды.
5. **Инфраструктура** — [`cybercity-blueprints`](https://github.com/TheCipherKeeper/cybercity-blueprints) + [`cybercity-agents`](https://github.com/TheCipherKeeper/cybercity-agents): узлы и агенты внутри сегментов.

## Принципы

- **События — единственный источник истины.** Состояние = проекция потока.
- **Сеть декларативна.** Один YAML описывает всё. K8s — его проекция.
- **Безопасность по умолчанию.** Сегменты изолированы, каналы — явные. OT не светится наружу.
- **LLM — помощник, не хозяин.** LLM пишет YAML, код валидирует, человек решает.
- **Воспроизводимость.** Один вход → один выход. Детерминированный режим, фиксированный порядок ключей.

## Как я работаю

(Выжимка из [`AGENTS.md`](https://github.com/TheCipherKeeper/cybercity-engine/blob/main/AGENTS.md).)

- По одной сущности за итерацию. Не «30 организаций сразу».
- Валидатор — единственный контракт правды. Каждое AI-изменение проходит через `go test`.
- LLM пишет YAML, не код. Код пишу я или валидатор.
- Коммиты и пуши — вручную. AI не пушит без явного одобрения.

## Статус

В активной разработке. v1.0 = 30 организаций / 95 сервисов / 3 сценария учений.
Дорожная карта — в [`master.md`](https://github.com/TheCipherKeeper/cybercity-engine/blob/main/master.md).

## Лицензия

- Код: [MIT](LICENSE)
- Документация: [CC BY 4.0](LICENSE-DOCS)
