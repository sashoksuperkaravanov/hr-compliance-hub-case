# Словарь данных: HR Compliance Hub

## Facility (производственная площадка)

| Поле | Тип | Обязательность | Описание |
|---|---|---|---|
| id | uuid | PK | Уникальный идентификатор площадки |
| name | varchar(200) | NOT NULL | Название площадки (например, «Производственная площадка №3, вахтовая») |
| is_shift_site | boolean | NOT NULL | Признак вахтового объекта; влияет на логистику доставки (см. точку отказа №2 AS-IS-процесса) |
| timezone | varchar(50) | NOT NULL | Часовой пояс площадки (IANA, например `Asia/Yakutsk`), используется при расчёте регуляторных сроков и SLA эскалации |

## Employee (сотрудник)

| Поле | Тип | Обязательность | Описание |
|---|---|---|---|
| id | uuid | PK | Уникальный идентификатор сотрудника |
| facility_id | uuid | FK → Facility | Площадка, к которой приписан сотрудник |
| full_name | varchar(300) | NOT NULL | ФИО сотрудника |
| snils | varchar(14) | NOT NULL | СНИЛС в формате `XXX-XXX-XXX XX`; проверяется контрольной суммой при валидации (UC2, FR-02) |
| inn | varchar(12) | NULLABLE | ИНН сотрудника (не всегда обязателен для формы ЕФС-1) |
| employment_type | varchar(30) | NOT NULL | Тип занятости: `regular` \| `shift` (вахтовый метод) |

## User (пользователь системы)

| Поле | Тип | Обязательность | Описание |
|---|---|---|---|
| id | uuid | PK | Уникальный идентификатор пользователя |
| full_name | varchar(300) | NOT NULL | ФИО пользователя |
| role | varchar(30) | NOT NULL | Роль для RBAC (NFR-08): `hr_specialist` \| `compliance_officer` \| `it_admin` |
| facility_id | uuid | FK → Facility, NULLABLE | Площадка пользователя (NULL для ролей с доступом ко всем площадкам, например `compliance_officer`) |

## HRDocument (кадровый документ)

| Поле | Тип | Обязательность | Описание |
|---|---|---|---|
| id | uuid | PK | Уникальный идентификатор документа |
| employee_id | uuid | FK → Employee | Сотрудник, к которому относится документ |
| document_type | varchar(30) | NOT NULL | Тип события: `hiring` \| `dismissal` \| `transfer` |
| status | varchar(40) | NOT NULL | Текущий статус, соответствует состояниям State Diagram: `draft`, `validating`, `sent_to_operator`, `transit_to_sfr`, `confirmed_by_sfr`, `retry_scheduled`, `requires_manual_intervention` |
| order_number | varchar(50) | NOT NULL | Номер приказа во внутренней HRM-системе |
| order_date | date | NOT NULL | Дата издания приказа, точка отсчёта регуляторного срока (NFR-01) |
| correlation_id | uuid | UNIQUE, NOT NULL | Ключ идемпотентности пакета (NFR-03); используется при сопоставлении webhook-ответов (NFR-05) |
| attempt_count | integer | NOT NULL, DEFAULT 0 | Текущее количество попыток отправки; ограничено `RetryPolicyConfig.max_attempts` (BR-01) |
| created_at | timestamptz | NOT NULL | Момент получения события от HRM (FR-01) |
| updated_at | timestamptz | NOT NULL | Момент последнего изменения статуса |

## DeliveryAttempt (попытка отправки)

| Поле | Тип | Обязательность | Описание |
|---|---|---|---|
| id | uuid | PK | Уникальный идентификатор попытки |
| document_id | uuid | FK → HRDocument | Документ, к которому относится попытка |
| attempt_number | integer | NOT NULL | Порядковый номер попытки (1–3 согласно BR-01) |
| sent_at | timestamptz | NOT NULL | Момент отправки пакета оператору ЭДО |
| response_received_at | timestamptz | NULLABLE | Момент получения ответа (webhook); NULL, если ответ ещё не получен |
| outcome | varchar(30) | NOT NULL | Результат попытки: `pending` \| `confirmed` \| `technical_failure` \| `content_rejected` |
| error_code | varchar(50) | NULLABLE | Код ошибки от оператора ЭДО/СФР, основа классификации технический/содержательный отказ (FR-06) |
| error_message | text | NULLABLE | Текстовое описание ошибки для отображения кадровику при эскалации (UC6) |

## StatusChangeLog (журнал изменений статуса)

| Поле | Тип | Обязательность | Описание |
|---|---|---|---|
| id | uuid | PK | Уникальный идентификатор записи |
| document_id | uuid | FK → HRDocument | Документ, к которому относится запись |
| old_status | varchar(40) | NOT NULL | Статус до изменения |
| new_status | varchar(40) | NOT NULL | Статус после изменения |
| changed_at | timestamptz | NOT NULL | Временная метка изменения |
| changed_by_user_id | uuid | FK → User, NULLABLE | Инициатор изменения; NULL, если изменение выполнено системой автоматически |
| reason | text | NULLABLE | Причина изменения (для ручных вмешательств кадровика/аудитора) |

**Важно (NFR-10):** таблица append-only. Записи никогда не обновляются и не удаляются, только добавляются — это прямое требование аудируемости.

## RetryPolicyConfig (конфигурация retry-политики)

| Поле | Тип | Обязательность | Описание |
|---|---|---|---|
| id | uuid | PK | Уникальный идентификатор версии конфигурации |
| max_attempts | integer | NOT NULL, DEFAULT 3 | Максимальное количество автоматических попыток (BR-01) |
| interval_1_minutes | integer | NOT NULL, DEFAULT 5 | Интервал перед 1-й повторной попыткой |
| interval_2_minutes | integer | NOT NULL, DEFAULT 30 | Интервал перед 2-й повторной попыткой |
| interval_3_minutes | integer | NOT NULL, DEFAULT 120 | Интервал перед 3-й повторной попыткой |
| updated_by_user_id | uuid | FK → User | Пользователь (роль `it_admin`), изменивший конфигурацию — UC9 |
| updated_at | timestamptz | NOT NULL | Момент последнего изменения |

**Примечание к дизайну:** таблица версионируется (каждое изменение — новая строка), а не перезаписывается. Это позволяет восстановить, какая политика действовала на момент конкретной попытки отправки, что важно при разборе инцидентов.

---

## Связь с FR/NFR

| Сущность/поле | Связанное требование |
|---|---|
| `Employee.snils` | FR-02 (валидация СНИЛС) |
| `Facility.is_shift_site` | Вахтовая специфика; учтена при разборе точки отказа №2 AS-IS-процесса |
| `HRDocument.correlation_id` | NFR-03, NFR-05 (идемпотентность) |
| `DeliveryAttempt.error_code` | FR-06 (классификация технический/содержательный отказ) |
| `StatusChangeLog` (вся таблица) | NFR-10 (аудируемость), FR-13 (журнал попыток) |
| `RetryPolicyConfig` | FR-12, UC9 |
