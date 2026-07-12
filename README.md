# CyberCity

[![Part of CyberCity](https://img.shields.io/badge/CyberCity-composition-blueviolet)](#)
[![License: MIT](https://img.shields.io/badge/code-MIT-green)](LICENSE)
[![Docs: CC BY 4.0](https://img.shields.io/badge/docs-CC%20BY%204.0-lightgrey)](LICENSE-DOCS)

**CyberCity** — модульный кибер-полигон: цифровой двойник городской ИТ/ОТ-
инфраструктуры для учений red/blue. Город моделируется как ориентированный граф
(организации → сервисы → связи достижимости), декларируется как код, рендерится
в артефакты, исполняется runtime. Не очередная CTF-машинка, а живой, наблюдаемый,
объяснимый город, где инциденты распространяются через достижимость (каждое
падение — наблюдённое коллектором событие). Всё крутится на вашем Proxmox / K8s,
без SaaS и внешней телеметрии.

Этот репозиторий — **хаб** программы: системные контракты и состав. Кода здесь
нет. Реализация каждого слоя — в отдельном репозитории (`cybercity-*`).
Методология построения — в
[`TheCipherKeeper/ai-project-template`](https://github.com/TheCipherKeeper/ai-project-template).

## Что в хабе

| Файл | Что |
|---|---|
| [`COMPOSITION.md`](COMPOSITION.md) | Состав программы: сервисы + интерфейсы + stub-таргеты, контракты, доверительная граница, ownership, потоки, статус реализации. Edge-реестр для verification «вниз». |
| [`CONVENTIONS.md`](CONVENTIONS.md) | Event envelope и кросс-сервисные конвенции общения (версионируется `CONVENTIONS@vN`). |
| [`AGENTS.md`](AGENTS.md) | Правила работы в хабе: приоритет доков, ветвление, можно/нельзя, версионирование контрактов, ADR-формат, нейминг, коммиты. |
| [`docker-compose.yml`](docker-compose.yml) | Системный compose: все сервисы + брокер (Redpanda). |
| [`adr/`](adr/) | Архитектурные решения (единый ADR-дом программы; индекс — `adr/README.md`). |

## Репозитории программы

Канонические состав, роли, стеки, контракты, пины и доверительная граница
описаны только в [`COMPOSITION.md`](COMPOSITION.md).

Профиль автора: [`TheCipherKeeper`](https://github.com/TheCipherKeeper/TheCipherKeeper) · [thecipherkeeper.github.io](https://thecipherkeeper.github.io).

## Разработка

```bash
git checkout main && git pull && git checkout -b feat/TASK-NNNN-<slug>
# правки COMPOSITION/CONVENTIONS/compose/adr; проверка перед коммитом
git commit -m "feat(conventions): ..." && git push   # PR в main
```

Прямой коммит в `main` запрещён; интеграция — только PR со squash merge.
Стабильная версия `vX.Y.Z` создаётся отдельной задачей человека. Правила —
[`AGENTS.md`](AGENTS.md).

### Запуск всей программы

```bash
cp .env.example .env            # заполнить per-сервис конф + BROKER_ADDR + CONVENTIONS_PIN
docker compose config           # проверить
docker compose up               # брокер + все сервисы (образы собраны заранее)
```

Команды запуска/сборки отдельного сервиса — в его репозитории (там код и
`Dockerfile`); в хабе их нет. Структура compose —
[`ai-project-template/docs/OPERATIONS.md`](https://github.com/TheCipherKeeper/ai-project-template/blob/main/docs/OPERATIONS.md).

## Лицензия

Код — [MIT](LICENSE), документация — [CC BY 4.0](LICENSE-DOCS). Полное правило о
лицензиях в каждом репозитории — [`AGENTS.md`](AGENTS.md) → *Лицензии*.
