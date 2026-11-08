# Web

[Spec] [web/001-service](../../specs/web/001-service.md), [web/002-admin](../../specs/web/002-admin.md).

## Role
Server-rendered UI (axum + askama + htmx) for customers: login, accounts, transactions, transfers, payees. Admin console under `/admin` (customers, accounts, funding, bulk transfers). Calls the gateway only.

## API
Pages: `/login`, `/accounts`, `/accounts/{id}`, `/transfer`, `/payees`, `/admin/*`. A transfer submit returns a "processing" fragment that polls (or uses SSE) until the state is `done`, `failed` or `reversed`.

## Data owned
None.

## Events
In: none (SSE from gateway for transfer results). Out: none.

## Interactions

```mermaid
flowchart LR
  B[browser] --> WEB[web]
  WEB -- "cookie, X-Admin-Token for /admin" --> GW[gateway]
  GW --> BE["accounts, ledger, transfers"]
```
