# ADR-0011: Выделить единый gateway и сохранить broker-only связность

## Status

Accepted

## Scope

Сквозное: `gateway`, `engine`, `manage`, `ui`, хаб.

## Context

UI обращался прямо к engine, а manage и engine документировали прямой
HTTP/gRPC control API. Это противоречит методологии: browser-facing API может
принадлежать только одному gateway, а service-to-service обмен идёт через
брокер.

## Decision

Создать `cybercity-gateway` как единственный browser-facing сервис. UI обращается
только к его `/v1/*` и `/ws`. Gateway публикует `city.commands`, потребляет
`city.state.updated` и поддерживает локальные читающие проекции. Engine больше
не предоставляет presentation API. Все команды manage к engine передаются
через топики Redpanda; прямой HTTP/gRPC control API удаляется из целевой модели.

## Consequences

- Появляется отдельный deployable и read model.
- UI и внутренние сервисы развязаны.
- Межсервисная сеть остаётся полностью broker-only.
- До завершения миграции gateway честно возвращает `501` на незавершённых
  endpoints.

## Alternatives considered

- Оставить engine gateway-сервисом — смешивает presentation и world-state.
- Разрешить control-plane исключение для HTTP/gRPC — нарушает общий инвариант
  методологии и усложняет проверку связности.

## Related

- `COMPOSITION.md`
- `CONVENTIONS.md`
- ADR-0001