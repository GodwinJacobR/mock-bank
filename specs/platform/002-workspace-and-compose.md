# Workspace and Compose

Status: draft

## Goal
A Cargo workspace with `common/` libraries and `services/` binaries, and a `docker compose up` that starts infrastructure and creates per-service DB roles and schemas.

## Scope (in)
- Root `Cargo.toml` workspace. Members: `common/*` (one library crate each: `money`, `errors`, `events`, `outbox`, `auth`, `telemetry`) and `services/*` (`accounts`, `ledger`, `gateway`, `web`, `seed`). Service crates may start as skeletons with a health endpoint.
- `docker-compose.yml`: `postgres` (`pgvector/pgvector:pg16`, port 5432, volume `pgdata`), `kafka` (`apache/kafka`, KRaft, port 9092, volume `kafkadata`, 3 partitions default), service containers, `seed` under profile `tools`.
- `.env.example`; no secrets committed.
- One shared cargo-chef `Dockerfile` (build arg `SERVICE`, path `services/$SERVICE`).
- Migrations layout and bootstrap of roles/schemas.

## Out of scope
- Library crate behaviour: see `common/001-money-and-errors`, `common/004-events-kafka`, `common/005-outbox`, `common/002-auth-claims`, `common/003-telemetry`.
- Redis, assistant, mcp-server, observability stack, kafka-ui, Qdrant.
- Service business logic (`accounts/*`, `ledger/*`, `gateway/001-service`, `web/001-service`, `seed/001-seed`).

## Design
**Layout**
```
Cargo.toml                 # [workspace] members = ["common/*", "services/*"]
common/{money,errors,events,outbox,auth,telemetry}/
services/{accounts,ledger,gateway,web,seed}/
migrations/{accounts,ledger}/
deploy/postgres/init.sql
```
- Crate names: `bank-money`, `bank-errors`, `bank-events`, `bank-outbox`, `bank-auth`, `bank-telemetry` for libraries; service crates are named `accounts`, `ledger`, `gateway`, `web`, `seed`.

**Postgres bootstrap** (`deploy/postgres/init.sql`, run on first start)
- One database `bank`; schemas `accounts`, `ledger`.
- Per schema two roles: `<svc>_owner` (owns the schema and its objects; used only to run migrations) and `<svc>_svc` (runtime role, limited grants defined in `accounts/002-schema` and `ledger/002-schema`). Passwords from env.
- Neither `_svc` role has any privilege on the other schema.

**Migrations**
- Naming: `migrations/<service>/NNNN_description.sql` (4-digit, zero-padded, increasing, e.g. `migrations/ledger/0001_init.sql`).
- Each service runs `sqlx::migrate!("../../migrations/<service>")` on startup connecting as `<svc>_owner`, then opens its runtime pool as `<svc>_svc`.

**Health**: every service uses the `common/003-telemetry` health helper (`/healthz`, `/readyz`).

**Compose**
- Build paths: `services/<name>` (e.g. `build: { context: ., args: { SERVICE: ledger } }`).
- Services depend on `postgres` and `kafka` healthchecks. Host ports: gateway 3000, web 8080; accounts/ledger internal only.
- `.env.example` keys: DB passwords per role, `JWT_SECRET`, `ADMIN_TOKEN`, `KAFKA_BROKERS`, `RUST_LOG`.

## Dependencies
- None (`platform/001-ci` recommended first).

## Acceptance criteria
- [ ] `cargo build --workspace` succeeds with all `common/*` and `services/*` members.
- [ ] `docker compose up` starts postgres and kafka healthy, and all service containers.
- [ ] Schemas `accounts` and `ledger` and their roles exist; a `_svc` role cannot write the other's schema.
- [ ] A `_svc` role cannot run DDL in its own schema (only `_owner` can).
- [ ] Rebuild after a source-only change reuses the cargo-chef dependency layer.

## Test plan
- Integration test (testcontainers) that bootstraps Postgres and checks role isolation.
- Manual: `docker compose up`, curl health endpoints.
