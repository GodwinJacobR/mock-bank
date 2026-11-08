# Gateway Admin API

Status: draft

## Goal
Admin-only gateway routes to create customers and accounts, fund accounts, and post transfers in bulk. This replaces any seed script: services contain no mock data.

## Scope (in)
- Routes under `/admin/*`, auth by header `X-Admin-Token` equal to `ADMIN_TOKEN` (single token; no admin login or password). Admin role JWT to accounts/ledger.
- Customers and accounts: proxy to accounts.
- Funding from treasury.
- Bulk transfers: sequential, per-item results, resubmit-safe.

## Out of scope
- Mock or random data generation in gateway routes (the random generator lives in the web admin layer, `web/002-admin`; it only produces a list of items).
- Any change to ledger or accounts business rules beyond admin role access.
- Atomic batches, bulk size cap.

## Design
**Routes** (admin only; missing or wrong token -> `401`; customer cookie never accepted)
- `POST /admin/customers` `{email, name, password}` -> accounts `POST /customers` -> `201`
- `GET /admin/customers` -> accounts `GET /customers`
- `POST /admin/accounts` `{customer_id, currency}` -> accounts `POST /accounts` -> `201`
- `GET /admin/accounts?customer_id=` -> accounts `GET /accounts` (admin lists all) composed with ledger `POST /balances/query` -> `[{id, customer_id, account_number, currency, status, balance_minor}]`
- `POST /admin/fund` `{to_account_number, amount_minor, currency}`, `Idempotency-Key` required -> transfers `POST /transfers` as admin from `bank:treasury:<CUR>`; returns `202 {transfer_id}`. The funding is final once the transfer is `done`.
- `POST /admin/bulk-transfers`, header `Idempotency-Key` = batch id (required, else 400), body `{items: [{from_account_id, to_account_number, amount_minor, currency, memo}]}`.

**Bulk transfers**
- Items processed sequentially, in order. No size cap.
- For item `i` (0-based) the gateway calls transfers `POST /transfers` as admin with `Idempotency-Key: {batch}-{i}`. Same path as a normal transfer: accepted with `202`, queued on `transfers.commands.v1`, run by the saga, `transfer.*` events on `transfers.events.v1`. Nothing is written directly to Kafka or SQL.
- Not atomic: each item is its own transfer; a failed item does not roll back or stop others. The batch response reports admission only; final outcomes (`done`, `failed`, `reversed`) come from `GET /transfers/{id}`.
- `503`/`429` with `Retry-After` from transfers is retried per item honouring `Retry-After` (backoff e.g. 250 ms, 500 ms, 1 s, 2 s, 4 s; then reported `failed`). `account_not_ready` is retried by the saga itself. Other errors are not retried.
- Response `200 {batch, items: [{index, status, transfer_id?, error?, message?}]}` where status is `accepted` (202, new transfer), `replayed` (transfers returned the existing transfer for the same key and body) or `failed` (rejected at admission or validation, error code passed through, e.g. `idempotency_conflict`, `overloaded`). `transfer_id` is set for `accepted` and `replayed`.
- Resubmit-safe: re-posting the same batch id with the same items replays `accepted` items (no duplicates) and retries `failed` ones. Same key with a different item body fails that item with `idempotency_conflict`. Item validation errors (missing field, non-positive amount) fail only that item.

**Errors**: `{error, message}` shape from `common/errors`.

## Dependencies
- `gateway/001-service`, `accounts/001-service`, `transfers/001-service`

## Acceptance criteria
- [ ] All `/admin/*` routes reject missing or wrong token; customer cookie cannot reach them.
- [ ] Admin can create and list customers and accounts (accounts list includes `customer_id` and balances).
- [ ] `POST /admin/fund` credits an account from treasury; repeating the same key creates one transfer.
- [ ] Bulk transfer without `Idempotency-Key` returns 400.
- [ ] Bulk item `i` uses transfers key `{batch}-{i}`; transfers appear on `transfers.events.v1` like normal transfers.
- [ ] Mixed batch returns per-item `created` / `failed`; failed items do not affect others.
- [ ] Re-posting the same batch returns `replayed` for earlier items and creates no duplicates.
- [ ] `account_not_ready` items are retried with backoff and succeed once the projection catches up.
- [ ] Admin can transfer from any account (admin role, `initiator_role = admin`); customer role cannot.
- [ ] No data generation code in gateway admin routes.

## Test plan
- Integration tests with accounts and ledger stubbed (wiremock): auth, proxying, key derivation, status mapping, retry/backoff, partial failure.
- End-to-end (compose): create customers and accounts, fund, run a bulk batch, resubmit it, check balances and zero-sum audit.
