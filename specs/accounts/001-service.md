# Accounts Service

Status: draft

## Goal
Own customers, accounts, payees and login sessions; publish account events; execute saga commands from transfers (debit, credit, undo) through the ledger. Keeps a read-only, eventually consistent balance projection built from `ledger.postings.v1`. The ledger stays the single source of truth for money and the authoritative overdraft check.

## Scope (in)
- Uses schema `accounts` (DDL, grants, migrations in `accounts/002-schema`).
- Publishes via `common/005-outbox`; JWT via `common/auth`; errors via `common/errors`; logging/health via `common/003-telemetry`.
- Account number generation.
- Argon2 login, Postgres-backed sessions (phase 1).
- Balance projection: consumes `ledger.postings.v1`, maintains `account_balances`, and serves `balance_minor` + `as_of` on account reads (see Balance projection).
- Saga command handling: consumes `accounts.commands`, dedupes by idempotency key, applies the posting through the ledger, publishes `accounts.replies`, acks the command only after the reply (see Saga commands).
- Authorization: a customer only accesses own data, enforced here from `X-Customer-Id` (signed internal JWT, `common/002-auth-claims`).

## Out of scope
- Authoritative balances, postings, overdraft enforcement, transfers (ledger). The projection is a read model only; nothing here decides whether money can move.
- Redis sessions (phase 2), KYC, email verification.
- Calling other services over HTTP, except ledger `POST /postings` when applying a saga command.
- Being called by transfers over HTTP other than for the read endpoints `GET /accounts/{id}` (start lookups, including the advisory balance check at admission) and `GET /accounts/resolve`. Money movement comes only through `accounts.commands`.

## Design

**Account number**: 12 digits, 11 random + Luhn check digit, retried on unique conflict.

**Internal endpoints** (called only by gateway; JWT required except where noted)
- `POST /customers` (admin) `{email, name, password}` -> `201 {id, email, name}`
- `POST /auth/login` (no JWT) `{email, password}` -> `200 {customer_id, session_id}`; bad creds `401`
- `GET /sessions/{id}` -> `200 {customer_id}` or `404` if missing/expired
- `DELETE /sessions/{id}` -> `204`
- `POST /accounts` `{currency}` -> `201 {id, account_number, currency, status}` (owner from JWT; admin may pass `customer_id`)
- `GET /accounts` -> `200 [{id, account_number, currency, status, balance_minor, as_of}]` (admin: all accounts with `customer_id`; optional `customer_id` filter). Balance fields are eventually consistent, see Balance projection.
- `GET /customers` (admin) -> `200 [{id, email, name}]`
- `GET /accounts/{id}` -> `200 {id, account_number, currency, status, balance_minor, as_of}` or `404` (other customer's account is `404`; admin/service sees any)
- `GET /accounts/resolve?account_number=` -> `200 {id, currency, status}` or `404`. Minimal fields only (no owner, name or email). Used by the transfers start lookups; any authenticated caller. Fixes no cross-customer lookup by name or email.
- `PATCH /accounts/{id}` `{status: "closed"}` -> `200`
- `POST /payees` `{name, account_number}` -> `201 {id, name, account_number}`
- `GET /payees` -> `200 [...]`; `DELETE /payees/{id}` -> `204`

**Events** (topic `accounts.events.v1`, key `account_id`, envelope from `common/004-events-kafka`)
- `AccountOpened` payload `{account_id, customer_id, account_number, currency}`
- `AccountClosed` payload `{account_id}`
- Each change calls `outbox::enqueue(tx, ...)` inside the same transaction as the account change; the service runs `outbox::run_relay` (see `common/005-outbox`). No relay logic is defined here.

**Saga commands** (consumer of `accounts.commands`, producer of `accounts.replies`; message formats in `common/006-saga`)
- Group `accounts-commands`; commands are keyed by `saga_id`. Command `{saga_id, step, cmd, account, amount, idempotency_key}` with `cmd` in `debit`, `credit`, `undo_debit`.
- Dedupe by `idempotency_key`: the key and the stored reply are recorded with the command result. A seen key re-sends the stored reply and applies nothing again.
- Otherwise, in order: call ledger `POST /postings` as `role = service` (debit: account to `bank:transit:<CUR>`; credit: transit to account; undo_debit: transit to the original account) using the command's `idempotency_key` as the posting key; record the result; publish the reply to `accounts.replies` (key = `saga_id`, payload `{saga_id, step, result, code}`, `result` `ok` or `failed`, `code` the business reason such as `insufficient_funds`, `account_closed`); then commit the command offset. The offset is committed only after the reply is published, so a crash redelivers the command and the dedupe makes it safe.
- Late action: a `debit` whose `debit:undo` key was already seen is rejected with reply `failed`, code `compensated`, and no posting. An `undo_debit` for a debit that was never applied is recorded and answered `ok` with no effect (and blocks a later debit of the same saga).
- Retriable ledger errors (`409 account_not_ready`, timeouts) are retried by the consumer without replying; business errors (`422`) become `failed` replies.

**Balance projection** (eventually consistent, read-only)
- Consumes `ledger.postings.v1` with the `common/004-events-kafka` runner (group `accounts-balances`; dedup on `event_id` via `processed_events`, handler in the runner's transaction). The topic key is the source account id; a posting touches two accounts that may sit on different partitions, so ordering is by per-account `version`, not by arrival (see `ledger/001-service` event payload).
- For each side of `PostingPosted` (`from_account`, `to_account`) whose id exists in `accounts.accounts`: `INSERT INTO account_balances ... ON CONFLICT (account_id) DO UPDATE SET balance_minor = EXCLUDED.balance_minor, last_version = EXCLUDED.last_version, as_of = EXCLUDED.as_of, updated_at = now() WHERE account_balances.last_version < EXCLUDED.last_version`. The posting carries the resulting balance and version, so apply is a set, not an add: duplicates and stale (older version) events change nothing, and out-of-order delivery converges. Accounts not in `accounts.accounts` (treasury, transit) are skipped.
- Only this consumer writes `account_balances`. No API handler writes it.
- Read: `balance_minor` and `as_of` come from a left join. `as_of` is the `posted_at` of the last applied posting (RFC3339). An account with no postings yet returns `balance_minor: 0, as_of: null` ("no activity").
- Consistency: the value can trail the ledger by the Kafka and consumer lag (normally well under a second, unbounded if the consumer is down). It is for display and advisory checks only. Clients must show `as_of` and never treat the value as authoritative; authoritative reads go to ledger `GET /accounts/{id}/balance` (admin/audit).
- Rebuild: replay `ledger.postings.v1` from earliest (`REPLAY_FROM_EARLIEST=true`). Because apply is version-guarded, replay without truncating is safe; to repair suspected drift, run `reset_consumer(pool, "accounts-balances")` (truncates `account_balances` and that consumer's `processed_events`) and replay. Requires infinite retention of `ledger.postings.v1`.
- Metrics: `accounts_balance_projection_lag_seconds` (now minus `occurred_at` of the last applied event), `consumer_lag{group="accounts-balances"}`, `events_dlq_total`. Alerts: projection lag above 30 s for 2 min (warn), above 5 min (page); consumer lag above 1000 for 2 min; any DLQ message on the topic.

## Dependencies
- `ledger/001-service` (`ledger.postings.v1` payload, `POST /postings`), `common/006-saga` (command and reply messages), `accounts/002-schema`, `common/001-money-and-errors`, `common/002-auth-claims`, `common/003-telemetry`, `common/004-events-kafka`, `common/005-outbox`

## Acceptance criteria
- [ ] Creating a customer stores an argon2 hash, never plaintext.
- [ ] Login with correct password returns a session; wrong password returns 401.
- [ ] Expired session returns 404.
- [ ] Opening an account yields a unique valid 12-digit number and one outbox row.
- [ ] Admin can list all customers and accounts; customer role cannot.
- [ ] Closing an account emits `AccountClosed`.
- [ ] Customer A cannot read or modify customer B's accounts or payees.
- [ ] `GET /accounts/resolve` returns only `{id, currency, status}` for a valid number and 404 otherwise.
- [ ] Adding the same payee account number twice for a customer is a 409.
- [ ] Every outbox row is eventually published to `accounts.events.v1` with the account id as key.
- [ ] Service writes only to schema `accounts`.
- [ ] A `PostingPosted` event updates `account_balances` for both accounts; `GET /accounts` and `GET /accounts/{id}` return `balance_minor` and `as_of`.
- [ ] Delivering the same event twice, or an older-version event after a newer one, leaves the balance unchanged.
- [ ] Events for the two sides of a posting arriving in either order converge to the ledger balance.
- [ ] Treasury and transit sides are skipped without error.
- [ ] No account endpoint writes `account_balances` (only the consumer does).
- [ ] An account with no postings returns `balance_minor: 0, as_of: null`.
- [ ] Wiping `account_balances` and replaying `ledger.postings.v1` from earliest reproduces the ledger balances.
- [ ] After a mixed workload, every projected balance equals ledger `balances` once consumer lag is zero.
- [ ] A `debit` command moves funds to transit via one ledger posting and publishes an `ok` reply on `accounts.replies` keyed by `saga_id`; an overdraft yields a `failed` reply with `insufficient_funds`.
- [ ] A command redelivered with the same `idempotency_key` re-sends the stored reply and creates no second posting.
- [ ] The command offset is committed only after the reply is published (a crash between them redelivers; no double effect).
- [ ] A `debit` whose `debit:undo` key was already seen is rejected (`failed`, `compensated`) with no posting.
- [ ] An `undo_debit` for a never-applied debit answers `ok` with no effect.
- [ ] Metric `accounts_balance_projection_lag_seconds` is exported and the alert rule fires when the consumer is stopped.

## Test plan
- Unit: account number Luhn, password hashing.
- Integration (testcontainers Postgres + Kafka): each endpoint, authorization isolation, outbox + relay publishes and survives a relay restart without losing rows.
- Projection: fabricated `PostingPosted` events for duplicate, stale, reordered, unknown-account and replay cases; end-to-end against a real ledger comparing balances at zero lag.
