# Web Admin Console

Status: draft

## Goal
Admin pages in the web UI to enter data (customers, accounts, funding, bulk transfers) instead of a seed script.

## Scope (in)
- Routes under `/admin` in `services/web`, htmx + askama like the rest of the UI.
- Web holds `ADMIN_TOKEN` in server env and adds `X-Admin-Token` on gateway calls; the browser never sees it. Single token auth only; no separate admin login or password. `/admin` is enabled only when `ADMIN_TOKEN` is set.
- Random transfer generator (runs in the web admin layer only).

## Out of scope
- Mock data in accounts/ledger/gateway routes; a seed binary.
- Direct calls to accounts or ledger (gateway only).
- Bulk size cap.

## Design
| Route | Page | Gateway calls |
|---|---|---|
| `GET /admin` | links, counts | `GET /admin/customers`, `GET /admin/accounts` |
| `GET /admin/customers`, `POST /admin/customers` | list; add form (email, name, password) | `GET|POST /admin/customers` |
| `GET /admin/accounts`, `POST /admin/accounts` | list (customer, number, currency, status, balance); add form (customer, currency) | `GET|POST /admin/accounts` |
| `POST /admin/fund` | fund an account from treasury (account, amount, currency); hidden idempotency key | `POST /admin/fund` |
| `GET /admin/bulk` | bulk transfer form | `GET /admin/accounts` |
| `POST /admin/bulk/preview` | parse input, show item table, nothing sent | none |
| `POST /admin/bulk` | submit with batch id; results table | `POST /admin/bulk-transfers` |
| `POST /admin/bulk/retry` | resubmit same batch id and items | `POST /admin/bulk-transfers` |

**Bulk transfer form**
- Input A: paste CSV (header `from_account_id,to_account_number,amount_minor,currency,memo`) or a JSON array of items.
- Input B: random generator with count, amount range (min/max), memo prefix and chosen accounts (multi-select; same-currency pairs only, source != destination). Randomness is used in the web admin module only and just produces items; the generated list is frozen into the preview before submit.
- Preview shows the items and validation errors; submit sends exactly the previewed items.
- Hidden `batch` id (uuid) generated when the preview is rendered; reused on resubmit and retry, so double clicks or refresh create no duplicates. A new preview gets a new id.
- Results table: index, status (`created`, `replayed`, `failed`), transfer id, error message. "Retry failed" resubmits the same batch id and items; already created items replay.
- Amounts are decimal strings converted to minor units server-side (no floats).
- Gateway `401` on admin routes shows a configuration error (bad `ADMIN_TOKEN`), not a login redirect.

## Dependencies
- `web/001-service`, `gateway/002-admin`

## Acceptance criteria
- [ ] Admin can add and list customers and accounts from the UI.
- [ ] Admin can fund an account from treasury; double submit creates one transfer.
- [ ] Bulk form accepts pasted CSV and JSON; preview shows items and errors without sending.
- [ ] Random generator produces `count` valid items within the amount range among the chosen accounts.
- [ ] Results table shows per-item `created` / `replayed` / `failed`; "Retry failed" creates no duplicates.
- [ ] The token never appears in HTML, cookies, JS or any response to the browser.
- [ ] No separate admin login exists; `ADMIN_TOKEN` is the only gate.
- [ ] Randomness exists only in the web admin module; accounts, ledger and gateway have none.
- [ ] Web makes HTTP calls only to the gateway.

## Test plan
- Unit tests: CSV/JSON parsing, amount conversion, generator constraints (count, range, currency match, source != destination).
- Integration tests with a stubbed gateway: each route, batch id reuse, retry, token sent to gateway and absent from responses.
- Manual e2e through `docker compose up`: create customers and accounts, fund, run a generated batch, retry failed.
