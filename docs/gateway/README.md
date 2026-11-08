# Gateway

[Spec] [gateway/001-service](../../specs/gateway/001-service.md), [gateway/002-admin](../../specs/gateway/002-admin.md).

## Role
The single public REST API. Authenticates users (session cookie or admin token), mints an internal JWT, routes to accounts, ledger and transfers, composes account pages.

## API (public)
- Auth: `POST /auth/login`, `POST /auth/logout`, `GET /me`.
- Accounts: `GET|POST /accounts`, `GET /accounts/{id}/transfers`, payees (`/payees`).
- Transfers: `POST /transfers` (`Idempotency-Key`, returns `202`), `GET /transfers/{id}`, SSE `GET /transfers/{id}/events`.
- Admin (`X-Admin-Token`): `/admin/*` (customers, accounts, fund, bulk transfers).
- `GET /healthz`, `GET /readyz`.

## Data owned
None in Postgres. Redis (phase 2): sessions, rate limit buckets, idempotency fast path, balance cache.

## Events
In: none. Out: none.

## Interactions

```mermaid
flowchart LR
  WEB[web] --> GW[gateway]
  GW -- "sessions, accounts, payees" --> ACC[accounts]
  GW -- "balances" --> LED[ledger]
  GW -- "POST /transfers, results, SSE" --> TRF[transfers]
  GW -. "phase 2" .-> R[(Redis)]
```
