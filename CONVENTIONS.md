# CyberCity — Конвенции

Кросс-репозиторные конвенции проекта CyberCity. Действуют во всех шести
репозиториях; каждый `AGENTS.md` ссылается сюда. Если локальное правило
репозитория противоречит этому файлу — побеждает этот файл (или заводится
новый ADR).

## Иерархия документов

От старшего к младшему:

1. **`cybercity/COMPOSITION.md`** — канон состава, контрактов,
   доверительной границы. Единый источник правды о том, что есть в проекте.
2. **Репозиторий `docs/adr/`** — архитектурные решения этого репозитория
   (статус `superseded` не имеет силы; `amended` — см. ссылку).
3. **`AGENTS.md`** — операционные правила работы в репозитории.
4. **`README.md`** — краткое описание и quick start.
5. **`docs/`** — внутренняя документация репозитория.
6. **Код, тесты, конфиги** — реализация принятых решений.

Сквозные решения (затрагивающие несколько репозиториев) живут в
[`cybercity/adr/`](adr/), не в репозиториях.

При противоречии побеждает старший. Любое расхождение — повод создать новый
ADR; старые ADR помечаем `superseded`, **не удаляем**.

## Язык

- Вся документация и ADR — **на русском**.
- Английский допустим только для: бейджей, идентификаторов кода, имён
  библиотек, значений поля `Status:` (`Accepted` / `Superseded` /
  `Amended`), технических заголовков таблиц.

## Casing файлов

- Нарративные документы — `UPPER.md` (`ARCHITECTURE.md`, `DEVELOPMENT.md`,
  `DATA_FLOW.md`, `VISION.md`, `CONVENTIONS.md`).
- ADR — `NNNN-kebab-case.md` (`0001-two-graph-architecture.md`), нумерация
  сквозная в пределах репозитория, с ведущими нулями.
- Канонические имена в нижнем регистре: `README.md`, `AGENTS.md`,
  `COMPOSITION.md`, `CONTRIBUTING.md`, `CHANGELOG.md`.

## Скелет репозитория

Одинаковый набор слотов в каждом репозитории (кроме хаба — у него свой состав):

```
cybercity-<repo>/
├── README.md            # краткая сводка + quick start + 3 бейджа; ссылки на хаб COMPOSITION и свой AGENTS/docs
├── AGENTS.md            # governance: иерархия, принципы репо, allow/deny агента, структура, цикл, язык
├── CONTRIBUTING.md      # тонкий указатель → docs/DEVELOPMENT.md
├── LICENSE              # MIT
├── LICENSE-DOCS         # CC BY 4.0, 12-строчный deed, в корне
├── docs/
│   ├── ARCHITECTURE.md  # как реализовано ЗДЕСЬ (внутренняя архитектура, слойность, код-структура)
│   ├── DEVELOPMENT.md   # build / lint / test / рецепты; раздел «Тестирование» обязателен
│   ├── DATA_FLOW.md     # внутренний поток данных/контрактов (честная заглушка, если пока нет)
│   ├── adr/
│   │   ├── README.md    # индекс ADR со статусами
│   │   └── NNNN-*.md
│   └── <репо-спец. доки>
└── <код>
```

Хаб `cybercity/` держит только системные документы: `README.md`,
`COMPOSITION.md`, `VISION.md`, `ARCHITECTURE.md`, `CONVENTIONS.md`, `adr/`,
`LICENSE`, `LICENSE-DOCS`. Кода нет.

## AGENTS.md — агент-слой

- В каждом репозитории в корне лежит `AGENTS.md` (vendor-neutral; читается и
  Claude Code, и другими агент-инструментами). `CLAUDE.md` не заводим.
- `.claude/` (планы, `settings.local.json`) — **вне governance**: это
  сессионное состояние, часто gitignored, не часть иерархии документов.
- Минимальное содержание `AGENTS.md`: ссылка на хаб `COMPOSITION.md` и
  `CONVENTIONS.md`; иерархия документов; ключевые принципы репозитория;
  «агенту МОЖНО / НЕЛЬЗЯ»; структура репозитория; рабочий цикл; правило
  языка.
- Коммиты, пуши, PR — делает человек, не агент.

## ADR — формат

- Отдельный файл `docs/adr/NNNN-kebab.md`.
- Заголовки: `# ADR-NNNN: <title>`, `## Status`, `## Context`, `## Decision`,
  `## Consequences`, `## Alternatives considered`, `## Related`.
- `Status`: `Accepted` | `Superseded` (со ссылкой на заменивший) |
  `Amended` (со ссылкой).
- `docs/adr/README.md` — индекс: номер, заголовок, статус, ссылка.
- Старые решения помечаем `Superseded`, не удаляем.

## README — контракт

- Три бейджа: `CyberCity-composition` (→ `TheCipherKeeper/cybercity`), MIT,
  CC BY 4.0.
- Краткая сводка (что это, стек, статус).
- Quick start.
- Ссылка на хаб `COMPOSITION.md` (канон) и на свой `AGENTS.md` + `docs/`.
- **Не** дублирует таблицу состава проекта — только ссылка на хаб.

## Состав проекта — единственный источник

6-репо таблица состава, контракты и доверительная граница описаны **только**
в `cybercity/COMPOSITION.md`. Ни один репозиторий не копирует её таблицей;
все ссылаются на хаб. При расхождении канон — в хабе.

## Нейминг

- Репозитории: `cybercity-<слайс>` (`cybercity-data`, `cybercity-engine`, …).
- Идентификаторы сущностей — kebab-case (`bank-web`, `hospital-db`).
- Топология: `id`, `org_id`, `network_id` — kebab-case; IP/CIDR — аллокатором
  из `cybercity-data`, воспроизводимо через `--seed`.
- Crate-имена коллектора: `ccc-*` (cyber city collector); бинарник
  `cybercity-collector`. Старые `ccna-*` / `cybercity-node-agent` сняты.

## Event envelope — кросс-репо контракт

Каноническая форма события, пересекающего границы репозиториев
(`collector → Kafka → engine`; `engine → WebSocket → ui`):

```json
{
  "event_id": "<uuid>",
  "parent_event_ids": ["<uuid>"],
  "correlation_id": "<scenario/incident>",
  "tick": 0,
  "timestamp": "<RFC3339>",
  "source_type": "engine|service|scenario|player|system|collector",
  "source_id": "<id>",
  "event_type": "SCAN|ATTACK|COMPROMISE|...",
  "target_id": "<topology node id | null>",
  "payload": {},
  "status": "pending|processed|failed|suppressed"
}
```

События `cybercity-collector` дополнительно оборачиваются в подписанный
конверт (Ed25519); Kafka — mTLS + ACL на продюсеров; гости до брокера не
достукиваются. Полный набор полей модели — в
[`cybercity-engine`/docs/MODELS.md](https://github.com/TheCipherKeeper/cybercity-engine/blob/main/docs/MODELS.md).

## Логирование

- Structured JSON, с `correlation_id` и `event_id` для сквозной трассировки.
- Уровни: `error`, `warn`, `info`, `debug`. Секреты в логи никогда.

## Коммиты и PR

- Conventional Commits: `feat:`, `fix:`, `docs:`, `refactor:`, `test:`,
  `chore:`, `adr:` (для ADR). Область опциональна: `feat(engine): ...`.
- Тело коммита — на русском; summary line — английский допустим.
- PR: кратко что и зачем, ссылка на ADR если есть, статус проверок
  (lint/test/build).

## Лицензии

- Код: MIT (`LICENSE`).
- Документация: CC BY 4.0 (`LICENSE-DOCS`, 12-строчный deed, в корне каждого
  репозитория).
- Бейджи в README ссылаются на эти файлы.

## LLM — помощник, не хозяин

- LLM пишет код/YAML/документы; валидаторы, тесты и линтеры решают.
- По одной сущности за итерацию, не «30 организаций сразу».
- Коммиты и пуши — вручную; агент не пушит без явного одобрения.