# Ledger Service

Status: draft

## Goal
Source of truth for money: double-entry postings, balances, idempotency, treasury and transit accounts, and a projection of accounts built from Kafka. The ledger no longer runs the transfer workflow; the transfers service (`transfers/001-service`) orchestrates debit and credit postings.

## Scope (in)
- Uses schema `ledger` (DDL, grants, invariants, system accounts in `ledger/002-schema`).
- Posting creation: one posting = two entries summing to zero, with overdraft rejection on the debited account.
- `account_projection` consumer for `accounts.events.v1`, built on the `common/004-events-kafka` consumer runner (dedup, retry, DLQ are the crate's job).
- Treasury accounts (`bank:treasury:<CUR>`) and transit accounts (`bank:transit:<CUR>`), created by migration.
- Publishes `ledger.postings.v1` via `common/005-outbox` (enqueue in the posting transaction, relay from the crate).

## Out of scope
- The transfer workflow: state machine, saga, compensation, queueing, results polling (`transfers/001-service`).
- FX / cross-currency postings, Redis cache.
- Customers, payees, account numbers (accounts service).
- Calling other services over HTTP.

## Design

**Endpoints** (internal; JWT with `X-Customer-Id`)
- `POST /postings` header `Idempotency-Key` (required), body `{from_account_id, to_account_id, amount_minor, currency, memo, transfer_ref, initiator_role}` -> `201 {id, from_account, to_account, amount_minor, currency, memo, transfer_ref, created_at}`
  - Caller role must be `service` (the transfers workers), else 403. Accounts are given by id; there is no account-number lookup here.
  - `initiator_role` (`customer` | `admin`) is the role of the original requester, copied by the transfers service. A treasury source requires `initiator_role = admin` (else 403), enforced here, not only in gateway or transfers.
- `GET /accounts/{id}/balance` -> `200 {account_id, balance_minor, currency}` (owner only)
- `POST /balances/query` `{account_ids: [...]}` -> `200 [{account_id, balance_minor, currency}]` (owner only; used by gateway to compose pages)
- `GET /accounts/{id}/postings?limit=&before=` -> `200 [{id, direction, counterparty, amount_minor, currency, memo, transfer_ref, created_at}]` (owner only). Postings against `bank:transit:<CUR>` are hidden from customer history; the gateway shows customer transaction history from the transfers service (`GET /accounts/{id}/transfers`).
- `GET /postings/{id}` -> `200` (admin/service only)

**Posting algorithm** (one DB transaction)
1. Resolve both accounts in `account_projection`; missing -> `409 account_not_ready` (retriable, same key safe). Closed destination or source (not for a reversal, see below) -> `422 account_closed`. Currency mismatch -> `422 currency_mismatch`.
2. Idempotency lookup: same key + same request hash -> return stored response; same key + different body -> `422 idempotency_conflict`.
3. Lock both `balances` rows in account-id order.
4. If the source is not treasury and balance < amount -> `422 insufficient_funds`. This is the authoritative overdraft check; the accounts pre-check in the transfers service is advisory only.
5. Insert posting, two entries (sum 0), update balances, store idempotency record and response, `outbox::enqueue`.
6. Commit, return 201.

Reversal postings (the transfers service sends `{transfer_id}-debit-undo`: transit to the original source) are ordinary postings. They skip the closed-account check on the destination so a debit can be returned even if the customer closed the account meanwhile.

**Transit accounts**: a transfer's debit posting moves funds customer to `bank:transit:<CUR>`; the credit posting moves transit to the destination. Each posting stays two entries summing to zero, so the zero-sum invariant needs no change.

**Events**
- Consumes `accounts.events.v1` with the `common/004-events-kafka` runner (group `ledger-projection`; handler runs in the runner's transaction, so `processed_events` dedup and offset commit are the crate's job): `AccountOpened` upserts projection and balance row (0); `AccountClosed` sets status.
- Publishes `PostingPosted` via `outbox::enqueue` to `ledger.postings.v1`, key = source `account_id`, payload `{posting_id, transfer_ref, from_account, to_account, amount_minor, currency, memo, posted_at}`. Used for cache invalidation. Transfer-level events (`transfer.*`) come from the transfers service, not the ledger.

## Dependencies
- `ledger/002-schema`, `common/001-money-and-errors`, `common/002-auth-claims`, `common/003-telemetry`, `common/004-events-kafka`, `common/005-outbox`. `accounts/001-service` for the events it consumes (can be tested with fabricated events).

## Acceptance criteria
- [ ] Every posting has two entries summing to zero; an audit query over all postings returns zero violations.
- [ ] Balances equal the sum of entries per account.
- [ ] Debit that would overdraw a non-treasury account returns 422 `insufficient_funds`; treasury may go negative.
- [ ] `POST /postings` with a customer or admin (non-`service`) role returns 403.
- [ ] A treasury source with `initiator_role` other than `admin` returns 403 even if gateway and transfers are bypassed.
- [ ] Unknown account returns 409 `account_not_ready`; after the event is consumed, retry with the same key succeeds.
- [ ] Cross-currency posting is rejected.
- [ ] Same key + same body returns the same response with one posting; same key + different body returns 422.
- [ ] A reversal posting succeeds when the original source account has since been closed.
- [ ] Concurrent opposing postings do not deadlock (ordered locks).
- [ ] Entries cannot be updated or deleted by the service role.
- [ ] Duplicate `accounts.events.v1` delivery has no extra effect.
- [ ] Wiping consumer offsets and projection, then replaying from earliest, rebuilds `account_projection`.
- [ ] Every posting appears on `ledger.postings.v1` (duplicates tolerated).
- [ ] Transit balance is zero after all transfers reach a final state.
- [ ] The ledger exposes no transfer workflow (no `POST /transfers`).

## Test plan
- Unit: request hashing, sign conventions.
- Integration (testcontainers Postgres + Kafka): all acceptance items above; concurrent-posting test; relay restart; projection replay.
