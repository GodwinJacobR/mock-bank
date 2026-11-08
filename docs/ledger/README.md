# Ledger

[Spec] [ledger/001-service](../../specs/ledger/001-service.md), [ledger/002-schema](../../specs/ledger/002-schema.md).

## Role
Source of truth for money: double-entry postings, balances, idempotency, treasury and transit accounts. The only authoritative overdraft check. Does not run the transfer workflow.

## API (internal)
- `POST /postings` (`Idempotency-Key`; `service` role only, called by transfers workers).
- `GET /accounts/{id}/balance`, `POST /balances/query` (authoritative reads for admin/audit via the gateway; customer pages read the accounts projection instead).
- `GET /accounts/{id}/postings`, `GET /postings/{id}`.

## Data owned
Schema `ledger`: postings, entries, balances, idempotency records, `account_projection`, processed events, system accounts (`bank:treasury:<CUR>`, `bank:transit:<CUR>`), outbox.

## Events
- In: `accounts.events.v1` (builds `account_projection`).
- Out: `ledger.postings.v1` (`PostingPosted` with `from/to_balance_after` and per-account `from/to_version`), via outbox. Consumed by the accounts balance projection.

## Interactions

```mermaid
flowchart LR
  TRF["transfers (saga: debit, credit, undo)"] -- "POST /postings, key id-step" --> LED[ledger]
  GW[gateway] -. "admin/audit balances" .-> LED
  LED --> PG[("Postgres: schema ledger")]
  K{{Kafka}} -- accounts.events.v1 --> LED
  LED -- "outbox: ledger.postings.v1" --> K
  K -- "ledger.postings.v1" --> ACC["accounts (balance projection)"]
```
