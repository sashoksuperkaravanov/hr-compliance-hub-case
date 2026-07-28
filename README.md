# HR Compliance Hub
Business &amp; System Analysis case study: event-driven integration service automating dismissal reporting to a government registry — BPMN, C4, UML, OpenAPI, Gherkin, full requirements traceability.

*[English version](README.en.md)*

Аналитическая и архитектурная документация интеграционного сервиса, устраняющего рассинхрон кадровых данных между внутренней HRM-системой и государственным реестром социального страхования (СФР).

> Учебный аналитический кейс Business Analyst / System Analyst. Компания-кейс («МеталлИнвестХолдинг») условна, но бизнес-проблема, нормативная база и архитектурные решения реалистичны и опираются на действующее законодательство (27-ФЗ, 125-ФЗ, 152-ФЗ) — см. [`docs/business-analysis/requirements.md`](docs/business-analysis/requirements.md).

## Бизнес-контекст

Компании со штатом 10 000+ сотрудников обязаны передавать сведения о трудовой деятельности в СФР по форме ЕФС-1 не позднее рабочего дня, следующего за кадровым событием (п. 2 ст. 11 Федерального закона № 27-ФЗ). При ручном процессе возникает четыре системные точки отказа — от невалидируемых опечаток в СНИЛС до отсутствия обратной связи об отклонении, обнаруживаемого спустя недели. Точки отказа разобраны в BPMN AS-IS-диаграмме и сведены в [`docs/business-analysis/traceability-matrix.md`](docs/business-analysis/traceability-matrix.md) (колонка «Точка отказа AS-IS»).

![BPMN TO-BE: процесс увольнения](docs/process-models/bpmn/dismissal-to-be.png)

*Процесс TO-BE — исходники: [AS-IS](docs/process-models/bpmn/dismissal-as-is.bpmn) / [TO-BE](docs/process-models/bpmn/dismissal-to-be.bpmn).*

## Решение

Выделенный интеграционный сервис (anti-corruption layer) между HRM и внешним контуром (Оператор ЭДО → СФР), с автоматической валидацией на входе, классификацией отказов на технические/содержательные и раздельной обработкой каждого типа (retry с фиксированными интервалами vs немедленная эскалация), обратной связью в реальном времени через вебхук.

## Возможности

- Автоматический приём кадровых событий из HRM (событийно, через шину сообщений)
- Валидация данных до отправки во внешние системы
- Формирование и отправка пакета оператору ЭДО с ЭЦП
- Retry-политика: 3 попытки, интервалы 5 мин / 30 мин / 2 часа — только для технических сбоев
- Немедленная эскалация кадровику при содержательных отклонениях (без бессмысленных повторов)
- Журнал статусов с RBAC-доступом для Compliance-офицера
- Настраиваемая retry-политика для ИТ-администратора

## Архитектурный подход

Выбран паттерн выделенного интеграционного сервиса вместо доработки статусной модели непосредственно в legacy HRM — это изолирует HRM от нестабильности и изменений внешнего контура (оператор ЭДО, СФР) и позволяет развивать retry/эскалацию независимо. Архитектура декомпозирована по C4-модели (Context → Component), где каждый компонент 1-в-1 соответствует границе конкретного Use Case (см. [`docs/architecture/`](docs/architecture/)).

![Context Diagram (C4 L1)](docs/architecture/context-diagram-c4-l1.png)

## Артефакты

| Уровень | Артефакт |
|---|---|
| Бизнес-процессы | [BPMN AS-IS](docs/process-models/bpmn/dismissal-as-is.bpmn) ([PNG](docs/process-models/bpmn/dismissal-as-is.png)) / [BPMN TO-BE](docs/process-models/bpmn/dismissal-to-be.bpmn) ([PNG](docs/process-models/bpmn/dismissal-to-be.png)) |
| Функциональные требования | [Use Case Diagram](docs/process-models/use-case-diagram.puml) ([PNG](docs/process-models/use-case-diagram.png)) · [Use Case Specification](docs/business-analysis/use-case-specification.md) · [User Stories (UC8/UC9)](docs/business-analysis/user-stories.md) |
| Архитектура | [Context Diagram (C4 L1)](docs/architecture/context-diagram-c4-l1.puml) ([PNG](docs/architecture/context-diagram-c4-l1.png)) · [Component Diagram (C4 L2)](docs/architecture/component-diagram-c4-l2.puml) ([PNG](docs/architecture/component-diagram-c4-l2.png)) |
| Поведение | [State Diagram](docs/process-models/state-diagram.puml) ([PNG](docs/process-models/state-diagram.png)) · [Sequence Diagram](docs/process-models/sequence-diagram.puml) ([PNG](docs/process-models/sequence-diagram.png)) |
| Данные | [ER-диаграмма](docs/architecture/er-diagram.puml) ([PNG](docs/architecture/er-diagram.png)) · [Словарь данных](docs/data/data-dictionary.md) |
| Требования | [FR/NFR](docs/business-analysis/requirements.md) · [Traceability Matrix](docs/business-analysis/traceability-matrix.md) |
| API | [OpenAPI 3.1](api/openapi.yaml) · [Обоснование контракта](api/api-contracts-overview.md) |
| Тестирование | [Тестовые сценарии (Gherkin)](docs/testing/test-scenarios.md) |

## Технологии и нотации

BPMN 2.0 · UML (Use Case / State / Sequence) · C4 Model · ER-моделирование · OpenAPI 3.1 · Gherkin

Технологический контекст решения (уровень понимания аналитика): event-driven архитектура, Kafka, PostgreSQL, mTLS, JWT/RBAC, усиленная квалифицированная ЭП.

## Структура проекта

```
docs/business-analysis/   — требования, use cases, user stories, трассировка
docs/process-models/      — BPMN, Use Case / State / Sequence диаграммы (+ PNG)
docs/architecture/        — C4 Context / Component, ER (+ PNG)
docs/data/                — словарь данных
docs/testing/             — тестовые сценарии
api/                      — OpenAPI-контракт и обоснование решений
```

## Как открыть диаграммы

PNG-превью каждой диаграммы уже лежат рядом с исходниками и открываются прямо на GitHub без дополнительных инструментов. Исходники нужны, только если хотите редактировать диаграмму:

- **`.puml`** (PlantUML, C4/UML) — расширение [PlantUML](https://marketplace.visualstudio.com/items?itemName=jebbs.plantuml) в VS Code, [plantuml.com/plantuml](https://www.plantuml.com/plantuml), или локально `plantuml *.puml`.
- **`.bpmn`** (BPMN 2.0 XML) — [bpmn.io](https://bpmn.io/) в браузере (drag-and-drop файла) или Camunda Modeler.
- **`.yaml`** (OpenAPI) — [Swagger Editor](https://editor.swagger.io/) или расширение Swagger Viewer в VS Code.

## Результаты

- 12 функциональных и 10 нефункциональных требований, полностью трассируемых от бизнес-целей до тестовых сценариев
- Согласованность между артефактами проверена и исправлена там, где расходилась, например, единая терминология статусов документа (Use Case Spec ↔ State Diagram) и единая правовая база для срока хранения логов (NFR-02) во всех документах.

## Об ограничениях

Это аналитический артефакт, не реализация. Тестовые сценарии специфицированы (Gherkin), но не автоматизированы и не выполнены, исполняемой системы нет. Условный технический долг и открытые вопросы (UC1/UC2/UC7 без отдельной спецификации, часть NFR без тестового покрытия) зафиксированы в [`docs/business-analysis/traceability-matrix.md`](docs/business-analysis/traceability-matrix.md) (раздел 4).
