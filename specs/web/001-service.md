# Web UI

Status: draft

## Goal
Server-rendered browser UI (htmx + askama) for the customer: login, accounts, transactions, transfers, payees.

## Scope (in)
- Crate `services/web` (axum + askama), port 8080.
- Pages: login, accounts and balances, transaction list, transfer form, payees.
- htmx for partial updates (transfer submit, payee add/remove, transaction list paging).
- Calls the gateway only; forwards the user's session cookie.

## Out of scope
- Assistant chat, approvals (later phases).
- Client-side frameworks, SPA, custom JS beyond htmx.
- Database access, calls to accounts or ledger.

## Design
| Route | Page | Gateway calls |
|---|---|---|
| `GET /login`, `POST /login` | email/password form; on success redirect `/accounts` and relay the session cookie | `POST /auth/login` |
| `POST /logout` | clear cookie, redirect `/login` | `POST /auth/logout` |
| `GET /accounts` | accounts with number, currency, balance | `GET /accounts` |
| `GET /accounts/{id}` | transaction list (direction, counterparty, amount, memo, date), "load more" via htmx | `GET /accounts/{id}/transfers` |
| `GET /transfer` | form: from account, to (payee dropdown or account number), amount, memo, hidden `idempotency_key` (uuid generated when the form is rendered) | `GET /accounts`, `GET /payees` |
| `POST /transfer` | submits with `Idempotency-Key` = hidden field; returns a "processing" fragment that follows the transfer until a final state | `POST /transfers`, then `GET /transfers/{id}` (htmx polling) or `GET /transfers/{id}/events` (SSE) |
| `GET /payees`, `POST /payees`, `DELETE /payees/{id}` | list, add, remove payees | `/payees` |

- Amount entered as decimal string, converted to minor units server-side (no floats); display formatted by currency.
- A resubmitted form (double click, refresh, retry) reuses the same hidden key, so no duplicate transfer. A fresh key is generated after a successful result.
- `POST /transfers` returns `202`: the UI shows "processing" (not success). The fragment polls every 1 s (or uses SSE) until `state` is `done` (success message, balances refresh), `failed` or `reversed` (friendly message from `failure_code`, e.g. `insufficient_funds`). If the user leaves the page, the transaction list shows the transfer with its state.
- `429`/`503` with `Retry-After` shown as "busy, retrying" and retried automatically once after `Retry-After`; with the same hidden key.
- Unauthenticated (gateway 401) redirects to `/login`.
- Askama templates in `services/web/templates/` with a base layout.

## Dependencies
- `gateway/001-service`, `platform/002-workspace-and-compose`

## Acceptance criteria
- [ ] A customer can log in and sees accounts with balances.
- [ ] Transaction list for an account shows transfers and pages with "load more".
- [ ] A transfer from the UI shows "processing", then success when `done`, and updates both balances.
- [ ] Submitting the same form twice (same hidden key) creates one transfer.
- [ ] Overdraft shows the failure message after processing and changes nothing.
- [ ] `503`/`429` shows a busy message honouring `Retry-After`.
- [ ] Payees can be added, listed, removed; payee dropdown is used in the transfer form.
- [ ] Logged-out access to any page redirects to `/login`.
- [ ] Web crate makes HTTP calls only to the gateway.

## Test plan
- Unit tests for amount parsing and formatting, template rendering.
- Integration tests with a stubbed gateway for each route, including idempotency key reuse and error rendering.
- Manual e2e through `docker compose up` with customers created via the admin console (`web/002-admin`).
