# Specs

Specs are grouped by area. Each folder holds the spec(s) for one area, numbered from 001 within the folder; `000-architecture.md` is the overall design.

Folders
- `platform/`: CI, Cargo workspace, Docker Compose, Postgres bootstrap.
- `common/`: reusable Rust library crates in `common/<name>/`, shared by services.
- `accounts/`, `ledger/`: service spec (`001-service.md`) and Postgres DDL (`002-schema.md`).
- `gateway/`, `web/`, `seed/`: one spec each.

## Changes
- Specs are grouped by area folder.
- Numbering restarts in each folder.
- New common library specs: events-kafka, outbox, money-and-errors, auth-claims, telemetry.
- Full DB schema specs for accounts and ledger.
- Code layout is now `common/` crates plus `services/`.

| Spec | Title | Depends on |
|---|---|---|
| [000-architecture](000-architecture.md) | Architecture | none |
| [platform/001-ci](platform/001-ci.md) | CI | none |
| [platform/002-workspace-and-compose](platform/002-workspace-and-compose.md) | Workspace and Compose | none |
| [common/001-money-and-errors](common/001-money-and-errors.md) | Money and errors crates | platform/002-workspace-and-compose |
| [common/002-auth-claims](common/002-auth-claims.md) | Auth claims crate | common/001-money-and-errors |
| [common/003-telemetry](common/003-telemetry.md) | Telemetry crate | platform/002-workspace-and-compose |
| [common/004-events-kafka](common/004-events-kafka.md) | Events (Kafka) crate | common/001-money-and-errors, common/003-telemetry |
| [common/005-outbox](common/005-outbox.md) | Outbox crate | common/004-events-kafka |
| [accounts/001-service](accounts/001-service.md) | Accounts service | accounts/002-schema, common/* |
| [accounts/002-schema](accounts/002-schema.md) | Accounts DB schema | platform/002-workspace-and-compose |
| [ledger/001-service](ledger/001-service.md) | Ledger service | ledger/002-schema, common/* |
| [ledger/002-schema](ledger/002-schema.md) | Ledger DB schema | platform/002-workspace-and-compose |
| [gateway/001-service](gateway/001-service.md) | Gateway | accounts/001-service, ledger/001-service |
| [web/001-service](web/001-service.md) | Web UI | gateway/001-service |
| [seed/001-seed](seed/001-seed.md) | Seed | gateway/001-service |

## Build order
1. `platform/001-ci` and `platform/002-workspace-and-compose` first.
2. `common/*` libraries (001-money-and-errors, 002-auth-claims, 003-telemetry, then 004-events-kafka, then 005-outbox).
3. `accounts/*` and `ledger/*` in parallel (002-schema first, then 001-service).
4. `gateway/001-service`.
5. `web/001-service` and `seed/001-seed` in parallel.
