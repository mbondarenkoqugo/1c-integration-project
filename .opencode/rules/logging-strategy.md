---
description: Positive logging strategy for 1C — when to write to the event log, which severity levels and category names to use, structured payload via `ДанныеЖурналаРегистрации`, secrets / PII bans. Complements the bans in `dev-standards-core.md §2 → "Forbidden Calls and Constructs"` and `dev-standards-architecture.md §3 → "Error Handling"`.
alwaysApply: false
---

# Logging Strategy

`dev-standards-core.md §2 → "Forbidden Calls and Constructs"` bans `ЗаписьЖурналаРегистрации` without an explicit task; `dev-standards-architecture.md §3 → "Error Handling"` bans empty `Попытка / Исключение`. This file is the **positive** companion: when logging *is* explicitly requested, this is how to do it.

## 1. When to log

| Scenario | Log? | Notes |
|---|---|---|
| Integrations — outbound HTTP / SOAP / queue request | **Yes** | Include endpoint, HTTP method, status code, response time, correlation id. Never include the request body if it contains PII or credentials. |
| Integrations — inbound webhook / scheduled exchange | **Yes** | Source, payload size, accepted / rejected, error reason. |
| Background jobs (`ДлительныеОперации`, `РегламентныеЗадания`) | **Yes** | Start / finish, success / failure, key parameters (anonymized). Necessary because the user has no UI feedback. |
| Posting errors (`ОбработкаПроведения` → `Отказ = Истина`) | **Yes** | Document, organization, the business reason; the user already saw the error, but the log is the only place where it survives. |
| Transactional rollback | **Yes** | Reason + document / operation key. |
| Standard CRUD on metadata via UI | **No** | The platform-level audit is enough. |
| Unhandled exceptions caught by an outer handler | **Yes** | Full `ПодробноеПредставлениеОшибки(ИнформацияОбОшибке())`. |
| Trace / debug output during development | **No** | Use the debugger / `ПоказатьЗначение`; remove before commit. See `systematic-debugging.md → Phase 4`. |

Rule of thumb: **log what cannot be reconstructed later**. Errors users see, integration boundaries, background work. Do not log mundane reads / writes that already leave a metadata trace.

## 2. Severity levels

`УровеньЖурналаРегистрации` has four values; treat them as a contract:

| Level | Use for |
|---|---|
| `Ошибка` (Error) | Operation failed and the system is in an inconsistent / pending-investigation state. Routes to monitoring. |
| `Предупреждение` (Warning) | Operation succeeded with a degraded path (retry succeeded, fallback used, optional dependency unavailable). Does not page on-call, but appears in summary reports. |
| `Информация` (Information) | Business-significant event that succeeded — completed integration, finished background job, audit-worthy state change. |
| `Примечание` (Note) | Diagnostic detail; rarely used in production code paths. Prefer omitting over emitting noise. |

Map them to your monitoring tier (Zabbix / ELK / Prometheus): typically `Ошибка` → alert, `Предупреждение` → digest, the rest → searchable only.

## 3. Event-category naming

`ЖурналРегистрации` events are filtered by category (event name). Use a consistent, hierarchical, dotted naming so that `ОтборЖурналаРегистрации` works predictably:

```
<Subsystem>.<Operation>.<Outcome>
```

Examples:

- `Интеграция.ОбменСCRM.Запрос`
- `Интеграция.ОбменСCRM.Ответ`
- `Интеграция.ОбменСCRM.Ошибка`
- `Проведение.РеализацияТоваровУслуг.КонфликтБлокировки`
- `ФоновоеЗадание.ОбновлениеКурсовВалют.Старт`
- `ФоновоеЗадание.ОбновлениеКурсовВалют.Финиш`

Reserved prefix `Debug.*` — **only** during active debugging, must be removed before commit (`verification-checklist.md → Soft gate A`). Never ship `Debug.*` to production.

## 4. Structured payload

Use `Структура` as the `Данные` argument of `ЗаписьЖурналаРегистрации` whenever the event has any context fields. Plain-string concatenated payloads are not parseable by downstream tooling.

```bsl
ДанныеСобытия = Новый Структура;
ДанныеСобытия.Вставить("Документ", СсылкаДокумента);
ДанныеСобытия.Вставить("Организация", СсылкаДокумента.Организация);
ДанныеСобытия.Вставить("Длительность_мс", Длительность);
ДанныеСобытия.Вставить("HTTPСтатус", ОтветHTTP.КодСостояния);

ЗаписьЖурналаРегистрации(
	"Интеграция.ОбменСCRM.Ответ",
	УровеньЖурналаРегистрации.Информация,
	Метаданные.Документы.РеализацияТоваровУслуг,
	СсылкаДокумента,
	"",
	ДанныеСобытия);
```

Fields:

- **`ИмяСобытия`** — the dotted category from §3.
- **`Уровень`** — from §2.
- **`Метаданные`** — the metadata object whose lifecycle the event belongs to. Pass it for posting / write events; omit (`Неопределено`) for cross-cutting events like scheduled-job lifecycle.
- **`ДанныеИсточник`** — the specific reference (document, catalog item). Lets `ОтборЖурналаРегистрации` filter to one entity in one click.
- **`Комментарий`** — short free-text. Use an empty string when `ДанныеСобытия` already carries everything.
- **`Данные`** — the `Структура`. Engine serializes via XDTO and lets downstream parsers read individual fields.

## 5. Error / exception logging

Inside a justified `Попытка / Исключение` block (e.g. transactional control, integration boundary), the error must be written **once** with the full description:

```bsl
Попытка
	ВыполнитьИнтеграционныйЗапрос();
Исключение

	ДанныеОшибки = Новый Структура;
	ДанныеОшибки.Вставить("Подробно", ПодробноеПредставлениеОшибки(ИнформацияОбОшибке()));
	ДанныеОшибки.Вставить("Запрос", СтрокаЗапроса);

	ЗаписьЖурналаРегистрации(
		"Интеграция.ОбменСCRM.Ошибка",
		УровеньЖурналаРегистрации.Ошибка,
		,
		,
		"Не удалось выполнить запрос к CRM.",
		ДанныеОшибки);

	ВызватьИсключение;

КонецПопытки;
```

Rules:

- **`ПодробноеПредставлениеОшибки(ИнформацияОбОшибке())`**, not `КраткоеПредставлениеОшибки()` — short representation drops the stack and the cause.
- **Always re-raise after logging at the integration boundary**, unless the contract is "swallow on this specific error code". Silent log-and-continue is the most common cause of phantom data inconsistencies.
- **Do not double-log.** If an inner handler already wrote a `Уровень.Ошибка` entry, the outer handler logs at most `Уровень.Предупреждение` ("the operation was aborted because <inner cause>") without re-dumping the same `Подробно`.

## 6. What MUST NOT go into the log

- **Credentials**: passwords, tokens, API keys, OAuth secrets, session cookies — even in `Подробно`. Always redact before logging.
- **PII at request-body granularity**: full passport / СНИЛС / payment card / email lists. Log a hash or a record id, not the value itself. Whether a given field is PII depends on the project's data-classification — when in doubt, do not log it.
- **Large blobs / attachments**. Log the metadata (size, content type, storage id), not the bytes.
- **The whole `ТаблицаЗначений`**. Log a summary (`КоличествоСтрок`, key columns), not the whole table.

`memory.md` is the canonical place to record project-specific PII classifications that affect logging — see `AGENTS.md → Project memory`.

## 7. Rotation and retention

- **`ЖурналРегистрации` is unbounded by default.** Configure rotation through `СерверныйКлиент` / Designer (`Администрирование → Журнал регистрации → Настройки`). For high-volume systems, set a separation period (`СократитьЖурналРегистрации` periodically via a regulated job) and archive older slices.
- **Off-host export.** For projects with monitoring infrastructure, periodically export via `ВыгрузитьЖурналРегистрации` to an external sink (ELK, Splunk, Kafka) and prune the platform-side log after. Building a real-time tail is out of scope for the platform — use the technological log if you need stream-grade telemetry.

## 8. Companion rules

| Concern | File |
|---|---|
| Ban on uninvited `ЗаписьЖурналаРегистрации` calls | `dev-standards-core.md §2 → "Forbidden Calls and Constructs"` |
| Ban on empty `Попытка / Исключение` | `dev-standards-architecture.md §3 → "Error Handling"` |
| Removing `Debug.*` log entries before commit | `verification-checklist.md → "Soft gate A"`, `systematic-debugging.md → "Phase 4"` |
| Background-job lifecycle logging | `platform-solutions.md §2 → "Long-running operations"` |
| Integration request / response logging | `integrations-add.md` |
