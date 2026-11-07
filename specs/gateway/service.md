# Gateway

Status: draft

## Goal
The single public REST API. Authenticates users, routes to accounts and ledger, and composes account pages.

## Scope (in)
- Public REST endpoints for web UI and seed.
- Session cookie auth (HttpOnly) backed by accounts' `sessions` table.
- Internal signed JWT (with customer id and role) on every call to accounts/ledger; `X-Customer-Id` header carries the customer id.
- Account page composition: accounts list + ledger balances.
- `Idempotency-Key` passthrough.
- Admin token auth for seed operations.

## Out of scope
- Redis sessions, rate limiting, idempotency fast path, balance cache (phase 2).
- Assistant and MCP routes.
- Business logic of accounts or ledger.

## Design
**Auth**
- `POST /auth/login` `{email, password}` -> `204` + `Set-Cookie: session=<id>; HttpOnly; SameSite=Lax; Path=/`. Calls accounts `/auth/login`.
- `POST /auth/logout` -> `204`, clears cookie.
- Each request: read cookie, look up via accounts `GET /sessions/{id}`, mint short-lived (60 s) JWT `{sub: customer_id, role: customer}`, call services with `Authorization: Bearer` and `X-Customer-Id`.
- Admin: header `X-Admin-Token` equal to `ADMIN_TOKEN` env gives role `admin` JWT (acts for the `customer_id` given in the request where needed). Never reachable via cookie.

**Public endpoints** (cookie auth unless noted)
- `GET /me` -> `{id, email, name}`
- `GET /accounts` -> `[{id, account_number, currency, status, balance_minor}]` (accounts list composed with ledger `POST /balances/query`)
- `POST /accounts` `{currency}` -> `201`
- `GET /accounts/{id}/transfers?limit=&before=` -> passthrough to ledger
- `POST /transfers` header `Idempotency-Key` (required, else 400), body `{from_account_id, to_account_number, amount_minor, currency, memo}` -> ledger response; status codes and error codes passed through unchanged (incl. `409 account_not_ready`).
- `GET|POST /payees`, `DELETE /payees/{id}` -> accounts passthrough
- Admin only (`X-Admin-Token`): `POST /admin/customers`, `POST /admin/accounts` (with `customer_id`), `POST /admin/fund` `{to_account_number, amount_minor, currency}` -> ledger transfer from treasury (Idempotency-Key required).
- `GET /healthz`, `GET /readyz`.

**Composition**: gateway calls accounts then ledger. If ledger fails, return `502`; do not return partial balances.

**Errors**: same `{error, message}` shape as `common/errors`. Missing/invalid session -> `401`.

## Dependencies
- `accounts/service`, `ledger/service`, `common/money-and-errors`, `common/auth-claims`, `common/telemetry`

## Acceptance criteria
- [ ] Login sets an HttpOnly cookie; wrong password gives 401.
- [ ] Unauthenticated requests to protected routes get 401.
- [ ] Internal services reject requests without a valid JWT.
- [ ] `GET /accounts` returns accounts with balances composed from both services.
- [ ] `Idempotency-Key` reaches the ledger unchanged; retry returns the same transfer.
- [ ] Missing `Idempotency-Key` on `POST /transfers` returns 400.
- [ ] Ledger errors (`409 account_not_ready`, `422 insufficient_funds`) are passed through with their codes.
- [ ] Admin endpoints reject missing or wrong token; customer cookie cannot reach them.
- [ ] Customer cannot see another customer's data.
- [ ] Gateway is the only component calling accounts and ledger.

## Test plan
- Integration tests with accounts and ledger stubbed (wiremock) for composition, passthrough, auth failures.
- End-to-end test (testcontainers or compose) of login, account list, transfer, retry.
