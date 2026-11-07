# 001c Accounts Service

Status: draft

## Goal
Own customers, accounts, payees and login sessions; publish account events. Knows no balances.

## Scope (in)
- Tables in schema `accounts`: `customers`, `accounts`, `payees`, `sessions`, `outbox`.
- Account number generation.
- Argon2 login, Postgres-backed sessions (phase 1).
- Outbox relay publishing `accounts.events.v1`.
- Authorization: a customer only accesses own data, enforced here from `X-Customer-Id` (signed internal JWT).

## Out of scope
- Balances, transfers (ledger).
- Redis sessions (phase 2), KYC, email verification.
- Calling other services over HTTP.

## Design
**Tables**
- `customers(id uuid pk, email unique, name, password_hash, created_at)`
- `accounts(id uuid pk, customer_id fk, account_number unique, currency char(3), status open|closed, created_at)`
- `payees(id uuid pk, customer_id fk, name, account_number, created_at, unique(customer_id, account_number))`
- `sessions(id text pk, customer_id fk, created_at, expires_at)`
- `outbox(id uuid pk, topic, key, payload jsonb, created_at, published_at null)`

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

**Events** (topic `accounts.events.v1`, key `account_id`, JSON envelope from `common`)
- `AccountOpened` payload `{account_id, customer_id, account_number, currency}`
- `AccountClosed` payload `{account_id}`
- Outbox row inserted in the same transaction as the account change. Relay polls `FOR UPDATE SKIP LOCKED`, publishes (`acks=all`, idempotent producer), sets `published_at`. Crash between publish and mark may duplicate.

## Dependencies
- 001b

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
