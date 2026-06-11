---
description: when writing code that integrates 1C with another system (HTTP services, REST, message queues, webhooks)
alwaysApply: false
---

# 1C Integrations with External Systems

Applies to integration code: HTTP services, REST clients, web services, file exchange, message queues, webhooks.

## 1. Before writing code

- Check whether a ready-made solution already exists in БСП via `ssl_search` (subsystems "Интернет-поддержка пользователей", "Обмен данными", "Получение файлов из Интернета", "Цифровая подпись"). The required scheme is often already implemented.
- Find existing integrations in the configuration via `templatesearch` and `search_code` (semantic mode, queries like "HTTP запрос", "отправка JSON", "парсинг ответа").
- Agree the contract with the user explicitly: method, URL/endpoint, payload format, authentication scheme, timeouts, retry policy, and logging.
- For EmplDocs / PA Docs integrations, use the product documentation at <https://padocs.empldocs.app/> as the authoritative external contract source before writing or changing requests, payloads, or authentication logic.

For the full MCP playbook see `tooling-playbooks.md → Integrations`.

## 2. Long-running and blocking operations

- Network calls are potentially long-running. Run all integration operations in the background through the БСП **"Long-running operations"** subsystem (`ДлительныеОперации.ВыполнитьФункцию`), not through a direct `ФоновыеЗадания` call. See `platform-solutions.md §2 → "Long-running operations"`.
- On the client — no synchronous HTTP calls; use `НачатьВыполнение*` or an async wrapper (template — `platform-solutions.md §8 → "External components on the thin client"`).

## 3. HTTP client

- Use platform `HTTPСоединение` / `HTTPЗапрос` or the БСП wrapper. `КомпонентаHTTPСервисы` and third-party COM objects are forbidden (see `dev-standards-architecture.md §3 → "Cross-Platform Compatibility"`).
- Connection timeout and read timeout MUST be set **explicitly** — use values from `.dev.env` or configuration constants, not magic numbers in code.
- Any response code different from the expected one MUST be turned into a meaningful exception with `ПодробноеПредставлениеОшибки(ИнформацияОбОшибке())` written to the event log. See `dev-standards-architecture.md §3 → "Error Handling"`.

## 4. Serialization and data contract

- JSON — via platform `ЧтениеJSON` / `ЗаписьJSON` (or the equivalent БСП helper if your БСП version provides one — verify the exact name with `ssl_search` / `docinfo` before use). Manual string assembly is forbidden.
- Numbers, dates and booleans must be validated separately: agree the date format with the receiving side (typically `ISO 8601`), specify decimal precision for numbers explicitly.
- For XML — `ЧтениеXML` / `ЗаписьXML` plus XSD validation when a schema is available. Manual string parsing is forbidden.

## 5. Security

- Credentials, tokens, API keys — only via **write-protected configuration constants** or the БСП "Безопасное хранение паролей" subsystem. Hardcoding is forbidden (`dev-standards-architecture.md §3 → "Security"`).
- Validate the token/session before each request; implement token refresh centrally.

## 6. Idempotency and retries

- Mutating requests must be idempotent on the 1C side: store the operation key in an information register and check status before resending.
- Retry policy: bounded number of attempts with exponential backoff. Infinite retry loops are forbidden.

## 7. Testing

- Verify the contract first manually (Postman, curl) on a test endpoint, then capture expected responses as examples in comments / documentation.
- For unit-level checks of parsing/serialization, write a minimal handler that does not depend on the network.

## 8. Documentation

For every new integration module record at the top (or in the metadata-object card): the external system, the contract (URL, method, format), the authentication scheme, the required roles, and a link to the requirements document.

Local out-of-1C prototyping (curl, Postman, ad-hoc scripts) is acceptable for contract debugging only. Production code stays in BSL — Python or other languages do not enter the repository.
