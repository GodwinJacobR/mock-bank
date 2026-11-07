# Money and Errors

Status: draft

## Goal
Two small library crates, `common/money` (`bank-money`) and `common/errors` (`bank-errors`), giving every service one money type and one error shape.

## Scope (in)
- `Money` and `Currency` types.
- `AppError` enum and its HTTP mapping.

## Out of scope
- FX or currency conversion; formatting for display (the web service does that).
- Any I/O.

## Design
**Money** (`common/money`)
- `Money { amount_minor: i64, currency: Currency }`. `Currency` is a 3-letter uppercase ISO code newtype, validated on construction and deserialization. No floats anywhere.
- `checked_add`, `checked_sub` return `Result<Money, MoneyError>`: overflow and currency mismatch are errors, never wraps.
- serde: `{ "amount_minor": 1000, "currency": "USD" }`. sqlx: `amount_minor` as `bigint`, `currency` as `char(3)` (via `Currency` encode/decode).

**Errors** (`common/errors`)
- `AppError` enum with a stable `code()` and an HTTP status:

| Variant / code | HTTP |
|---|---|
| `validation` | 400 |
| `unauthorized` | 401 |
| `forbidden` | 403 |
| `not_found` | 404 |
| `account_not_ready` (retriable, same Idempotency-Key safe) | 409 |
| `already_exists` (e.g. duplicate payee) | 409 |
| `insufficient_funds`, `account_closed`, `currency_mismatch`, `idempotency_conflict` | 422 |
| `upstream` (dependency failed) | 502 |
| `internal` (details logged, not returned) | 500 |

- `IntoResponse` (axum) renders `{ "error": "<code>", "message": "..." }`. Conversions from `sqlx::Error` (unique violation to `already_exists`, otherwise `internal`) and `MoneyError`.
- Gateway passes service errors through unchanged by decoding the same body into `AppError`.

## Dependencies
- `platform/workspace-and-compose`.

## Acceptance criteria
- [ ] Money serde round-trips; invalid currency code is rejected.
- [ ] Overflow and cross-currency add/sub return errors.
- [ ] Every `AppError` variant maps to the status above with the documented body.
- [ ] `internal` never leaks the underlying message.

## Test plan
- Unit tests for Money arithmetic edge cases (i64 max/min), serde, and error to response mapping.
