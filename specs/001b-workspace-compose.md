# 001b Workspace and Compose

Status: draft

## Goal
A Cargo workspace, shared `common` crate and a `docker compose up` that starts infrastructure and creates per-service DB roles and schemas.

## Scope (in)
- Root `Cargo.toml` workspace with `crates/{common,accounts,ledger,gateway,web,seed}` (service crates may start as skeletons with a health endpoint).
- `crates/common`: money type, errors, event envelope, auth claims, tracing setup.
- `docker-compose.yml`: `postgres` (`pgvector/pgvector:pg16`, port 5432, volume `pgdata`), `kafka` (`apache/kafka`, KRaft, port 9092, volume `kafkadata`, 3 partitions default), service containers, `seed` under profile `tools`.
- `.env.example`; no secrets committed.
- One shared cargo-chef `Dockerfile` (build arg `CRATE`).
- Migrations layout and bootstrap of roles/schemas.
- Health endpoints on every service.

## Out of scope
- Redis, assistant, mcp-server, observability stack, kafka-ui, Qdrant.
- Service business logic (001c-001g).

## Design
**common**
- `Money { amount_minor: i64, currency: Currency }`; `Currency` is a 3-letter ISO code newtype. No floats. Checked add/sub; serde as `{ "amount_minor": 1000, "currency": "USD" }`.
- `AppError` enum mapped to HTTP via `IntoResponse`; body `{ "error": "<code>", "message": "..." }`. Codes include `not_found`, `validation`, `unauthorized`, `forbidden`, `insufficient_funds`, `account_not_ready`, `idempotency_conflict`.
- `EventEnvelope<T> { event_id: Uuid, type, occurred_at (RFC3339), schema_version: u32, aggregate_id, payload: T }` (JSON).
- `Claims { sub (customer id), iss, exp, role: customer|admin }`; sign/verify helpers for the internal JWT (HS256, secret from env).
- `init_tracing()`: `tracing-subscriber` JSON to stdout, `RUST_LOG` filter.

**Postgres bootstrap** (`deploy/postgres/init.sql`, run on first start)
- One database `bank`; schemas `accounts`, `ledger`; roles `accounts_svc`, `ledger_svc`, each owning/limited to its own schema (password from env).

**Migrations**
- `migrations/accounts/` and `migrations/ledger/`; each service runs `sqlx::migrate!` for its folder on startup using its own role.

**Health**
- `GET /healthz` returns `200 {"status":"ok"}`; `GET /readyz` also checks DB (and Kafka where used).

**Compose**
- Services depend on `postgres` and `kafka` healthchecks. Host ports: gateway 3000, web 8080; accounts/ledger internal only.
- `.env.example` keys: DB passwords per role, `JWT_SECRET`, `ADMIN_TOKEN`, `KAFKA_BROKERS`, `RUST_LOG`.

## Dependencies
- None (001a recommended first).

## Acceptance criteria
- [ ] `cargo build --workspace` succeeds.
- [ ] `docker compose up` starts postgres and kafka healthy, and all service containers.
- [ ] Schemas `accounts` and `ledger` and their roles exist; a role cannot write the other's schema.
- [ ] `/healthz` returns 200 on every service; `/readyz` fails when DB is down.
- [ ] Money serde round-trips; overflow is an error, not a wrap.
- [ ] JWT signed by one crate verifies in another; expired or tampered token is rejected.
- [ ] Rebuild after a source-only change reuses cargo-chef dependency layer.

## Test plan
- Unit tests for `Money`, error mapping, envelope serde, JWT.
- Integration test (testcontainers) that bootstraps Postgres and checks role isolation.
- Manual: `docker compose up`, curl health endpoints.
