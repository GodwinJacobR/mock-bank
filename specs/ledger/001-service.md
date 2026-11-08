# Ledger Service

Status: draft

## Goal
Source of truth for money: double-entry transfers, balances, idempotency, treasury accounts, and a projection of accounts built from Kafka.

## Scope (in)
- Uses schema `ledger` (DDL, grants, invariants, treasury rows in `ledger/002-schema`).
- Transfer creation with double-entry postings and overdraft rejection.
- `account_projection` consumer for `accounts.events.v1`, built on the `common/004-events-kafka` consumer runner (dedup, retry, DLQ are the crate's job).
- Treasury accounts (`bank:treasury:<CUR>`, created by migration), admin-only origination.
- Publishes `ledger.transfers.v1` via `common/005-outbox` (enqueue in the transfer transaction, relay from the crate).

## Out of scope
- FX / cross-currency transfers, Redis cache.
- Customers, payees, account numbers (accounts service).
- Calling other services over HTTP.

## Design

**Endpoints** (internal; JWT with `X-Customer-Id`)
- `POST /transfers` header `Idempotency-Key` (required), body `{from_account_id, to_account_number, amount_minor, currency, memo}` -> `201 {id, from_account, to_account, amount_minor, currency, memo, created_at}`
  - Caller must own the source account (else 404). Treasury source requires admin role (else 403), enforced here.
- `GET /accounts/{id}/balance` -> `200 {account_id, balance_minor, currency}` (owner only)
- `POST /balances/query` `{account_ids: [...]}` -> `200 [{account_id, balance_minor, currency}]` (owner only; used by gateway to compose pages)
- `GET /accounts/{id}/transfers?limit=&before=` -> `200 [{id, direction, counterparty, amount_minor, currency, memo, created_at}]`
- `GET /transfers/{id}` -> `200` (party only)

**Transfer algorithm** (one DB transaction)
1. Resolve source and destination in `account_projection`; missing -> `409 account_not_ready` (retriable, same key safe). Closed -> `422 account_closed`. Currency mismatch -> `422 currency_mismatch`.
2. Idempotency lookup: same key + same request hash -> return stored response; same key + different body -> `422 idempotency_conflict`.
3. Lock both `balances` rows in account-id order.
4. If source is not treasury and balance < amount -> `422 insufficient_funds`.
5. Insert transfer, two entries (sum 0), update balances, store idempotency record and response, `outbox::enqueue`.
6. Commit, return 201.

**Events**
- Consumes `accounts.events.v1` with the `common/004-events-kafka` runner (group `ledger-projection`; handler runs in the runner's transaction, so `processed_events` dedup and offset commit are the crate's job): `AccountOpened` upserts projection and balance row (0); `AccountClosed` sets status.
- Publishes `TransferPosted` via `outbox::enqueue` to `ledger.transfers.v1`, key = source `account_id`, payload `{transfer_id, from_account, to_account, amount_minor, currency, memo, posted_at}`.

## Dependencies
- `ledger/002-schema`, `common/001-money-and-errors`, `common/002-auth-claims`, `common/003-telemetry`, `common/004-events-kafka`, `common/005-outbox`. `accounts/001-service` for the events it consumes (can be tested with fabricated events).

## Acceptance criteria
- [ ] Every transfer has two entries summing to zero; an audit query over all transfers returns zero violations.
- [ ] Balances equal the sum of entries per account.
- [ ] Overdraft returns 422 `insufficient_funds`; treasury may go negative.
- [ ] Non-admin cannot originate from treasury (403) even if gateway is bypassed.
- [ ] Unknown destination or source returns 409 `account_not_ready`; after the event is consumed, retry with the same key succeeds.
- [ ] Cross-currency transfer is rejected.
- [ ] Same key + same body returns the same response with one transfer; same key + different body returns 422.
- [ ] Concurrent opposing transfers do not deadlock (ordered locks).
- [ ] Entries cannot be updated or deleted by the service role.
- [ ] Duplicate `accounts.events.v1` delivery has no extra effect.
- [ ] Wiping consumer offsets and projection, then replaying from earliest, rebuilds `account_projection`.
- [ ] Every posted transfer appears on `ledger.transfers.v1` (duplicates tolerated).

## Test plan
- Unit: request hashing, sign conventions.
- Integration (testcontainers Postgres + Kafka): all acceptance items above; concurrent-transfer test; relay restart; projection replay.
