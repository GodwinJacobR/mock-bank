# 001f Web UI

Status: draft

## Goal
Server-rendered browser UI (htmx + askama) for the customer: login, accounts, transactions, transfers, payees.

## Scope (in)
- Crate `crates/web` (axum + askama), port 8080.
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
| `POST /transfer` | submits with `Idempotency-Key` = hidden field; returns result fragment | `POST /transfers` |
| `GET /payees`, `POST /payees`, `DELETE /payees/{id}` | list, add, remove payees | `/payees` |

- Amount entered as decimal string, converted to minor units server-side (no floats); display formatted by currency.
- A resubmitted form (double click, refresh, retry) reuses the same hidden key, so no duplicate transfer. A fresh key is generated after a successful result.
- Errors from the gateway (`insufficient_funds`, `account_not_ready`, etc.) shown as friendly messages; `account_not_ready` tells the user to retry.
- Unauthenticated (gateway 401) redirects to `/login`.
- Askama templates in `crates/web/templates/` with a base layout.

## Dependencies
- 001b, 001e

## Acceptance criteria
- [ ] Seed user can log in and sees accounts with balances.
- [ ] Transaction list for an account shows transfers and pages with "load more".
- [ ] A transfer from the UI updates both balances.
- [ ] Submitting the same form twice (same hidden key) creates one transfer.
- [ ] Overdraft shows an error message and changes nothing.
- [ ] Payees can be added, listed, removed; payee dropdown is used in the transfer form.
- [ ] Logged-out access to any page redirects to `/login`.
- [ ] Web crate makes HTTP calls only to the gateway.

## Test plan
- Unit tests for amount parsing and formatting, template rendering.
- Integration tests with a stubbed gateway for each route, including idempotency key reuse and error rendering.
- Manual e2e through `docker compose up` with seeded data.
