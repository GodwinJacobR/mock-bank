# 001g Seed

Status: draft

## Goal
Populate a running bank with repeatable dummy data through the public API only.

## Scope (in)
- Crate `crates/seed`, runnable as `cargo run -p seed` or `docker compose run seed` (profile `tools`).
- Creates customers, accounts, payees (optional), treasury funding, and random transfers.
- Fixed RNG seed; deterministic `Idempotency-Key`s; safe to re-run.

## Out of scope
- Direct SQL or Kafka access.
- Vector/embedding or assistant data.
- Bulk load testing (phase 5).

## Design
**Flags** (with env/default): `--customers` (default 5), `--accounts-per-customer` (2), `--transfers` (50), `--seed` (42), `--gateway-url` (default `http://gateway:3000`), `--admin-token` (env `ADMIN_TOKEN`).

**Flow** (all via gateway, admin token for admin calls)
1. Generate customer names/emails with the faker crate from `StdRng::seed_from_u64(seed)`; emails derived deterministically (e.g. `user{n}@example.test`). All share one dev password.
2. `POST /admin/customers`; an existing email (409) is treated as success.
3. `POST /admin/accounts` per customer (account existence checked first via customer's session, so re-runs add none).
4. Fund each account with `POST /admin/fund` (treasury transfer), key `seed-{seed}-fund-{i}`.
5. Wait/retry on `409 account_not_ready` (backoff, same key) until the ledger has projected the account.
6. Random transfers between seeded accounts (same currency, amount within balance), logged in as the customer; key `seed-{seed}-xfer-{i}`.
7. Print summary and the dev password (e.g. `Seed users password: <password>`), plus the list of seed emails.

**Determinism**: every random choice comes from the seeded RNG, in a fixed order; transfer `i` is always the same parameters, so a replay hits the ledger's idempotency and creates nothing.

## Dependencies
- 001e (needs the full stack); 001f not required.

## Acceptance criteria
- [ ] `docker compose run seed` creates the requested customers, accounts and funded balances.
- [ ] Re-running with the same flags creates no new customers, accounts or transfers.
- [ ] Same `--seed` yields identical data in a fresh database; different `--seed` differs.
- [ ] Idempotency keys follow `seed-{seed}-{kind}-{index}`.
- [ ] `409 account_not_ready` is retried and eventually succeeds.
- [ ] Seed uses HTTP to the gateway only.
- [ ] Output includes the dev password.
- [ ] A seed user can log in via the UI.

## Test plan
- Unit tests: key derivation, RNG determinism (same seed, same plan).
- Integration test against the compose stack (or testcontainers): run twice and compare row counts; check ledger zero-sum audit.
