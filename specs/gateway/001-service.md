# Gateway

Status: draft

## Goal
The single public REST API. Authenticates users, routes to accounts, ledger and transfers. Account pages need only accounts (balances come from its projection).

## Scope (in)
- Public REST endpoints for the web UI.
- Session cookie auth (HttpOnly) backed by accounts' `sessions` table.
- Internal signed JWT (with customer id and role) on every call to accounts/ledger/transfers; `X-Customer-Id` header carries the customer id.
- Account list with balances: passthrough to accounts, which returns `balance_minor` and `as_of` from its eventually consistent projection. No ledger call.
- `Idempotency-Key` passthrough.
- Transfer submission (`202`) and result routes (poll, SSE) proxied to the transfers service; overload responses (`429`/`503` + `Retry-After`) passed through.
- Admin token auth for admin operations (see `gateway/002-admin`).

## Out of scope
- Redis sessions, per-user/IP rate limiting (`429` + `Retry-After`), idempotency fast path, balance cache (phase 2, optional; the accounts projection already removes the ledger hop).
- The transfer workflow and queue (`transfers/001-service`).
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
- `GET /accounts` -> `[{id, account_number, currency, status, balance_minor, as_of}]` (accounts passthrough; balances are eventually consistent, `as_of` is when the balance was last updated from the ledger, `null` if no postings yet)
- `POST /accounts` `{currency}` -> `201`
- `GET /accounts/{id}/transfers?limit=&before=` -> passthrough to transfers (includes in-flight transfers with `state`)
- `POST /transfers` header `Idempotency-Key` (required, else 400), body `{from_account_id, to_account_number, amount_minor, currency, memo}` -> transfers `POST /transfers`: `202 {transfer_id, status: "PENDING"}` with `Location: /transfers/{id}`. The result is not known yet. Error codes and `Retry-After` from transfers pass through unchanged (`400`, `422 idempotency_conflict`, `422 insufficient_funds` (advisory early reject, no transfer created), `429 too_many_in_flight`, `503 overloaded`).
- `GET /transfers/{id}` -> transfers passthrough: `{id, status, state, failure_code?, ...}` (`state`: `processing` | `done` | `failed` | `reversed`).
- `GET /transfers/{id}/events` -> SSE proxy to transfers (no buffering; closes after a final state).
- `GET|POST /payees`, `DELETE /payees/{id}` -> accounts passthrough
- Admin only (`X-Admin-Token`): `/admin/*` routes, specified in `gateway/002-admin`.
- `GET /healthz`, `GET /readyz`.

**Backpressure**: when Redis rate limits are on (phase 2), exceeding the per-user or per-IP bucket returns `429` with `Retry-After` before any call to transfers. Admission limits (backlog, consumer lag) are enforced by transfers and surface as `503`/`429` with `Retry-After`.

**Balances**: `GET /accounts` calls accounts only; there is no composition and no `502` path from a ledger failure. The ledger is down-tolerant for reads. Ledger `POST /balances/query` is used only by admin/audit routes (`gateway/002-admin`).

**Errors**: same `{error, message}` shape as `common/errors`. Missing/invalid session -> `401`.

## Dependencies
- `accounts/001-service` (incl. balance projection), `ledger/001-service` (admin/audit reads only), `transfers/001-service`, `common/001-money-and-errors`, `common/002-auth-claims`, `common/003-telemetry`

## Acceptance criteria
- [ ] Login sets an HttpOnly cookie; wrong password gives 401.
- [ ] Unauthenticated requests to protected routes get 401.
- [ ] Internal services reject requests without a valid JWT.
- [ ] `GET /accounts` returns accounts with `balance_minor` and `as_of` from accounts alone; with the ledger stopped it still returns `200`.
- [ ] `GET /accounts` makes no call to the ledger (wiremock ledger sees zero requests).
- [ ] `422 insufficient_funds` from the transfers admission pre-check reaches the client unchanged and no transfer exists for that key.
- [ ] `Idempotency-Key` reaches the transfers service unchanged; retry returns the same `transfer_id`.
- [ ] `POST /transfers` returns `202` with a `transfer_id`; `GET /transfers/{id}` and the SSE route return the progress and final state.
- [ ] `503`/`429` with `Retry-After` from transfers (or the gateway rate limit) reaches the client unchanged.
- [ ] Missing `Idempotency-Key` on `POST /transfers` returns 400.
- [ ] Transfers errors are passed through with their codes; business failures found by the saga (`insufficient_funds` at the ledger debit, `account_closed`) appear as the final transfer state (`failure_code`), not as the POST response; the only business reject at the POST is the advisory early `422 insufficient_funds`.
- [ ] Admin endpoints reject missing or wrong token; customer cookie cannot reach them.
- [ ] Customer cannot see another customer's data.
- [ ] Gateway is the only user-facing caller of accounts, ledger and transfers (the transfers workers call accounts and ledger for the saga).

## Test plan
- Integration tests with accounts, ledger and transfers stubbed (wiremock) for balance passthrough (ledger unused), passthrough, auth failures.
- End-to-end test (testcontainers or compose) of login, account list, transfer (202 then poll to `DONE`), retry.
