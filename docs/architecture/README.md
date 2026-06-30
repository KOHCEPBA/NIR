# Архитектурная документация

## Подход

Комплект построен как легковесная документация в стиле **arc42**, дополненная:

- **C4 Model** для system context и container views;
- Mermaid sequence/state diagrams для runtime view;
- Architecture Decision Records для значимых решений и отвергнутых альтернатив;
- contract-first описанием REST API через OpenAPI;
- risk-driven документами security, observability, deployment и testing;
- вертикальным release roadmap с трассировкой требований.

Это не буквальное заполнение шаблона arc42 и не TOGAF-процесс. Используется практичный **arc42-inspired + C4 + ADR + contract-first** подход, подходящий для эволюционной разработки приложения.

## Соответствие arc42

| Представление arc42 | Артефакт проекта |
|---|---|
| Introduction, goals, constraints | `01-requirements-and-scope.md` |
| Context and scope | C4 Context в `02-system-design.md` |
| Solution strategy, building blocks | `02-system-design.md`, `03-data-model.md` |
| Runtime view | `07-dynamic-and-event-design.md` |
| Deployment view | `08-deployment-and-delivery.md` |
| Cross-cutting concepts | API, security, observability и testing документы |
| Architectural decisions | `docs/adr` |
| Quality requirements | quality scenarios в документе 01, документы 05, 06 и 09 |
| Risks and technical debt | документы 05 и 10 |
| Glossary | `GLOSSARY.md` |

## Документы

| № | Архитектурный артефакт | Содержание |
|---|---|---|
| 1 | [Требования и границы MVP](01-requirements-and-scope.md) | роли, функции, ограничения и открытые вопросы |
| 2 | [Системный дизайн](02-system-design.md) | контекст, модули и взаимодействия |
| 3 | [Модель данных](03-data-model.md) | владение данными, таблицы и целостность |
| 4 | [Проектирование API](04-api-design.md) | REST, STOMP, ошибки и конкурентный доступ |
| 5 | [Обзор безопасности](05-security-review.md) | угрозы, контрмеры и открытые риски |
| 6 | [Наблюдаемость и эксплуатация](06-observability-and-operations.md) | telemetry, alerts, backup и runbooks |
| 7 | [Динамическая и событийная архитектура](07-dynamic-and-event-design.md) | sequence-сценарии, фоновые события и отказоустойчивость |
| 8 | [Развертывание и поставка](08-deployment-and-delivery.md) | среды, образы, scaling и pipeline |
| 9 | [Тестирование и качество](09-testing-and-quality.md) | уровни тестов и release gates |
| 10 | [Целевая архитектура и план реализации](10-architecture-review-and-implementation-plan.md) | релизы, открытые решения, epics/tasks и трассировка |

## Связанные артефакты

- [OpenAPI](../api/openapi.yaml) является точкой входа в REST-контракт.
- [ADR index](../adr/README.md) содержит принятые архитектурные решения.
- [Глоссарий](GLOSSARY.md) фиксирует предметные и технические термины.
- JSON Schema межпроцессных событий должны храниться рядом с модулем-владельцем после начала реализации.

## Правила актуализации

Изменение требования обновляет связанные системный дизайн, контракт, модель данных и тестовый сценарий. Значимое решение с альтернативами оформляется ADR. Breaking change REST или принятого межпроцессного контракта требует явной версии и плана миграции consumers.
