# Accounts Service

Status: draft

## Goal
Own customers, accounts, payees and login sessions; publish account events. Knows no balances.

## Scope (in)
- Uses schema `accounts` (DDL, grants, migrations in `accounts/schema`).
- Publishes via `common/outbox`; JWT via `common/auth`; errors via `common/errors`; logging/health via `common/telemetry`.
- Account number generation.
- Argon2 login, Postgres-backed sessions (phase 1).
- Authorization: a customer only accesses own data, enforced here from `X-Customer-Id` (signed internal JWT, `common/auth-claims`).

## Out of scope
- Balances, transfers (ledger).
- Redis sessions (phase 2), KYC, email verification.
- Calling other services over HTTP.

## Design

**Account number**: 12 digits, 11 random + Luhn check digit, retried on unique conflict.

**Internal endpoints** (called only by gateway; JWT required except where noted)
- `POST /customers` (admin) `{email, name, password}` -> `201 {id, email, name}`
- `POST /auth/login` (no JWT) `{email, password}` -> `200 {customer_id, session_id}`; bad creds `401`
- `GET /sessions/{id}` -> `200 {customer_id}` or `404` if missing/expired
- `DELETE /sessions/{id}` -> `204`
- `POST /accounts` `{currency}` -> `201 {id, account_number, currency, status}` (owner from JWT; admin may pass `customer_id`)
- `GET /accounts` -> `200 [{id, account_number, currency, status}]`
- `GET /accounts/{id}` -> `200` or `404` (other customer's account is `404`)
- `PATCH /accounts/{id}` `{status: "closed"}` -> `200`
- `POST /payees` `{name, account_number}` -> `201 {id, name, account_number}`
- `GET /payees` -> `200 [...]`; `DELETE /payees/{id}` -> `204`

**Events** (topic `accounts.events.v1`, key `account_id`, envelope from `common/events-kafka`)
- `AccountOpened` payload `{account_id, customer_id, account_number, currency}`
- `AccountClosed` payload `{account_id}`
- Each change calls `outbox::enqueue(tx, ...)` inside the same transaction as the account change; the service runs `outbox::run_relay` (see `common/outbox`). No relay logic is defined here.

## Dependencies
- `accounts/schema`, `common/money-and-errors`, `common/auth-claims`, `common/telemetry`, `common/events-kafka`, `common/outbox`

## Acceptance criteria
- [ ] Creating a customer stores an argon2 hash, never plaintext.
- [ ] Login with correct password returns a session; wrong password returns 401.
- [ ] Expired session returns 404.
- [ ] Opening an account yields a unique valid 12-digit number and one outbox row.
- [ ] Closing an account emits `AccountClosed`.
- [ ] Customer A cannot read or modify customer B's accounts or payees.
- [ ] Adding the same payee account number twice for a customer is a 409.
- [ ] Every outbox row is eventually published to `accounts.events.v1` with the account id as key.
- [ ] Service writes only to schema `accounts`.

## Test plan
- Unit: account number Luhn, password hashing.
- Integration (testcontainers Postgres + Kafka): each endpoint, authorization isolation, outbox + relay publishes and survives a relay restart without losing rows.
