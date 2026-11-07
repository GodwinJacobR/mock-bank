# Tasks

One row per task. Each task has a spec in `specs/`, a branch `feat/<id>-<name>`, and a PR.
Status: todo | in progress | done

| ID  | Task                                         | Spec | Branch | PR | Status |
|-----|----------------------------------------------|------|--------|----|--------|
| 001a | CI: GitHub Actions running cargo fmt, clippy and tests on every PR | - | - | - | todo |
| 001 | Phase 1: accounts, ledger, gateway, web UI (htmx), Postgres, Kafka + outbox, seed script | - | - | - | todo |
| 002 | Phase 2: Redis (sessions, cache, rate limit, idempotency) + hardening (DLQ, relay-crash tests) | - | - | - | todo |
| 003 | Phase 3: RAG over transactions with vector DB | -   | -      | -  | todo   |
| 004 | Phase 4: AI agent with tools and MCP server  | -    | -      | -  | todo   |
| 005 | Phase 5: load testing and observability      | -    | -      | -  | todo   |

## Project v2 (future)

| ID  | Task | Spec | Branch | PR | Status |
|-----|------|------|--------|----|--------|
| v2-01 | gRPC between services | - | - | - | todo |
| v2-02 | Avro/Protobuf events with a schema registry | - | - | - | todo |
| v2-03 | Leptos UI | - | - | - | todo |
| v2-04 | Qdrant swap behind VectorStore trait | - | - | - | todo |
| v2-05 | Kafka log compaction for accounts events | - | - | - | todo |
| v2-06 | Voyage AI embeddings provider | - | - | - | todo |
