# Traceability Matrix — финальная версия

**Статус:** единый файл-источник истины по трассировке бизнес-целей, требований, компонентов, API и тест-покрытия для проекта HR Compliance Hub.

---

## 1. Бизнес-цели → KPI

| Бизнес-цель | KPI | Целевое значение | База (экспертная оценка) |
|---|---|---|---|
| Снижение штрафов за нарушение сроков подачи в СФР | KPI-1 | −90% (со 120–180 до ≤12–18 эпизодов/год) | ~3 600 событий/год, 3–5% с нарушением срока |
| Сокращение цикла «подписание → подтверждение СФР» | KPI-2 | С 5–7 дней до ≤1 рабочего дня | Задержка ручного экспорта AS-IS: 2–5 дней |
| Снижение доли ручных расследований рассинхрона | KPI-3 | С ~15% до ≤2% | Отраслевой ориентир 10–20% при ручном вводе |
| Повышение прозрачности статуса для кадровой службы | KPI-4 | 100% событий с автоматическим статусом (сейчас ~0%) | Исходный профиль компании, раздел «Бизнес-цели» |

---

## 2. Use Case → требования → компонент → API → тест-покрытие

| UC | Название | Use Case Spec | FR | NFR | Точка отказа AS-IS | Компонент (C4 L2) | API-эндпоинт | User Story | Acceptance Criteria |
|---|---|---|---|---|---|---|---|---|---|
| UC1 | Получить кадровое событие | ❌ Не написан (Must, долг — см. п.4) | FR-01 | NFR-01 | №2 | Event Ingestion Service | — (Kafka, вне OpenAPI) | — | ❌ Нет |
| UC2 | Валидировать данные | ❌ Не написан (Must, долг) | FR-02, FR-03 | — | №1 | Validation Service | — | — | ❌ Нет (частично покрыт Gherkin UC3/5 косвенно) |
| UC3 | Сформировать и отправить пакет | ✅ [`use-case-specification.md`](use-case-specification.md) | FR-04 | NFR-01, NFR-02, NFR-03, NFR-07 | №2 | Dispatch Service | — (исходящий вызов к оператору, вне контракта Hub, см. [`api-contracts-overview.md`](../../api/api-contracts-overview.md), раздел 1) | — | ✅ Gherkin в [`use-case-specification.md`](use-case-specification.md) |
| UC4 | Получить статус от СФР | ✅ [`use-case-specification.md`](use-case-specification.md) | FR-06, FR-07 | NFR-04, NFR-05, NFR-09 | №3, №4 | Status Webhook Receiver | `POST /v1/webhooks/edo-status` | — | ✅ Gherkin в [`use-case-specification.md`](use-case-specification.md) |
| UC5 | Повторная отправка (retry) | ✅ [`use-case-specification.md`](use-case-specification.md) (extend UC3) | FR-05 | NFR-01, NFR-03 | №2 | Retry Scheduler | — | — | ✅ Gherkin в [`use-case-specification.md`](use-case-specification.md) |
| UC6 | Эскалация кадровику | ✅ [`use-case-specification.md`](use-case-specification.md) (extend UC4) | FR-08 | NFR-06 | №4 | Escalation Notifier | — (канал не определён, открытый вопрос №1) | — | ✅ Gherkin в [`use-case-specification.md`](use-case-specification.md) |
| UC7 | Опубликовать событие подтверждения | ❌ Не написан (Should, долг) | FR-09 | — | — | Event Publisher | — (Kafka) | — | ❌ Нет |
| UC8 | Просмотреть журнал статусов | ⚠️ Не как отдельный UC Spec, покрыт через User Stories (альтернативный формат, не пробел) | FR-11, FR-13 | NFR-08, NFR-10 | — | Journal API | `GET /v1/documents`, `/{id}`, `/{id}/attempts`, `/{id}/status-log` | US-01, US-02 | ✅ Gherkin в [`user-stories.md`](user-stories.md) |
| UC9 | Настроить retry-политику | ⚠️ Покрыт через User Stories | FR-12 | — | — | Retry Policy Config Service | `GET/PUT /v1/config/retry-policy` | US-03, US-04 | ✅ Gherkin в [`user-stories.md`](user-stories.md) |
| UC10 | Уведомление об ошибке валидации | ⚠️ Не выделен в отдельный UC по решению MoSCoW-приоритизации, часть альтернативного потока UC2/UC3 | FR-03 | — | №1 | Validation Service | — | — | ⚠️ Не оформлен отдельным Gherkin-сценарием (минорный пробел) |

**Легенда:** ✅ выполнено · ⚠️ выполнено альтернативным способом или частично · ❌ не выполнено (открытый пробел)

---

## 3. NFR → где проверяется

| NFR | Требование (кратко) | Проверяется в |
|---|---|---|
| NFR-01 | Первая попытка ≤60 сек | Не покрыто тест-сценарием (нефункциональное, требует нагрузочного теста, вне Gherkin) |
| NFR-02 | Хранение логов (архивное право) | Не покрыто тест-сценарием на этом этапе |
| NFR-03 | Идемпотентность retry | Косвенно — Gherkin UC3/5 ([`use-case-specification.md`](use-case-specification.md)) |
| NFR-04 | Обработка webhook ≤5 сек p95 | Не покрыто (требует нагрузочного теста) |
| NFR-05 | Идемпотентность webhook | ✅ Gherkin UC4 ([`use-case-specification.md`](use-case-specification.md)) + `WebhookAck.duplicate` в OpenAPI |
| NFR-06 | Эскалация ≤15 мин | Не покрыто тест-сценарием (требует замера времени, не чистый Gherkin) |
| NFR-07 | Контроль сертификата ЭЦП | ⚠️ Частично — новая ветка «содержательная ошибка при отправке» в Sequence Diagram, но отдельного Gherkin нет (см. раздел 4) |
| NFR-08 | RBAC | ✅ Gherkin US-01, US-04 ([`user-stories.md`](user-stories.md)) |
| NFR-09 | Доступность 99.5% | Не покрыто (SLA-мониторинг, не функциональный тест) |
| NFR-10 | Append-only журнал | ✅ Gherkin US-02 ([`user-stories.md`](user-stories.md)) |

---

## 4. Открытые пробелы (честно, без сокрытия)

1. **UC1, UC2, UC7 — нет Use Case Specification.** Планировалось написать их следом за консолидацией требований; это пока не выполнено. UC7 обнаружен как дополнительный пробел только сейчас, при сборке этой матрицы.
2. **Ветка «сбой самой отправки оператору ЭДО»** добавлена в BPMN/Sequence, но **не имеет собственного Gherkin-сценария** — критерии приёмки для неё пока не формализованы, хотя диаграммы обновлены.
3. **UC10** не имеет отдельного Gherkin, хотя логика реализована в BPMN/Sequence.
4. **6 из 10 NFR не имеют тестового покрытия** в принципе. Это ожидаемо: нагрузочные/SLA-требования не проверяются Gherkin-сценариями, но это должно быть явно зафиксировано в плане нагрузочного тестирования, которого пока не существует как артефакта.

Эти пробелы напрямую формируют объём тестовых сценариев следующего шага; раздел 4 закрывается частично там же.
