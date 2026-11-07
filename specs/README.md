# Specs

Specs are grouped by area. Each folder holds the spec(s) for one area; `000-architecture.md` is the overall design.

Folders
- `platform/`: CI, Cargo workspace, Docker Compose, Postgres bootstrap.
- `common/`: reusable Rust library crates in `common/<name>/`, shared by services.
- `accounts/`, `ledger/`: service spec (`service.md`) and Postgres DDL (`schema.md`).
- `gateway/`, `web/`, `seed/`: one spec each.

| Spec | Title | Depends on |
|---|---|---|
| [000-architecture](000-architecture.md) | Architecture | none |
| [platform/ci](platform/ci.md) | CI | none |
| [platform/workspace-and-compose](platform/workspace-and-compose.md) | Workspace and Compose | none |
| [common/money-and-errors](common/money-and-errors.md) | Money and errors crates | platform/workspace-and-compose |
| [common/auth-claims](common/auth-claims.md) | Auth claims crate | common/money-and-errors |
| [common/telemetry](common/telemetry.md) | Telemetry crate | platform/workspace-and-compose |
| [common/events-kafka](common/events-kafka.md) | Events (Kafka) crate | common/money-and-errors, common/telemetry |
| [common/outbox](common/outbox.md) | Outbox crate | common/events-kafka |
| [accounts/schema](accounts/schema.md) | Accounts DB schema | platform/workspace-and-compose |
| [accounts/service](accounts/service.md) | Accounts service | accounts/schema, common/* |
| [ledger/schema](ledger/schema.md) | Ledger DB schema | platform/workspace-and-compose |
| [ledger/service](ledger/service.md) | Ledger service | ledger/schema, common/* |
| [gateway/service](gateway/service.md) | Gateway | accounts/service, ledger/service |
| [web/service](web/service.md) | Web UI | gateway/service |
| [seed/seed](seed/seed.md) | Seed | gateway/service |

## Build order
1. `platform/ci` and `platform/workspace-and-compose` first.
2. `common/*` libraries (money-and-errors, auth-claims, telemetry, then events-kafka, then outbox).
3. `accounts/*` and `ledger/*` in parallel (schema first, then service).
4. `gateway/service`.
5. `web/service` and `seed/seed` in parallel.
