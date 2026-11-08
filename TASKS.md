# Tasks

Single source of truth. Every PR adds or updates its row here.
Status: todo | in progress | done

| Milestone | Done/Total |
|-----------|------------|
| M1 Core | 0/8 |
| M2 Events and Redis (Kafka) | 0/1 |
| M3 RAG over transactions with vector DB | 0/1 |
| M4 AI agent with tools and MCP server | 0/1 |
| M5 Load testing and observability | 0/2 |
| Chore | 0/1 |

## M1 – Core (accounts, ledger, Postgres, web UI, admin console)

| ID | Task | Spec | Branch | PR | Status |
|----|------|------|--------|----|--------|
| T-001 | Accounts service and schema | [accounts](specs/accounts/001-service.md), [schema](specs/accounts/002-schema.md) | - | - | todo |
| T-002 | Ledger service and schema | [ledger](specs/ledger/001-service.md), [schema](specs/ledger/002-schema.md) | - | - | todo |
| T-003 | Gateway service | [gateway](specs/gateway/001-service.md) | - | - | todo |
| T-004 | Web UI | [web](specs/web/001-service.md) | - | - | todo |
| T-005 | Admin console (gateway admin API, web /admin, no seed data) | [gateway](specs/gateway/002-admin.md), [web](specs/web/002-admin.md) | feat/admin-console-spec | - | in progress |
| T-006 | Platform CI | [ci](specs/platform/001-ci.md) | - | - | todo |
| T-007 | Workspace and compose | [workspace](specs/platform/002-workspace-and-compose.md) | - | - | todo |
| T-008 | Common libs (money/errors, auth claims, telemetry, events, outbox) | [common](specs/common/) | - | - | todo |

## M2 – Events and Redis (Kafka)

| ID | Task | Spec | Branch | PR | Status |
|----|------|------|--------|----|--------|
| T-009 | Kafka events and Redis | - | - | - | todo |

## M3 – RAG over transactions with vector DB

| ID | Task | Spec | Branch | PR | Status |
|----|------|------|--------|----|--------|
| T-010 | RAG over transactions with vector DB | - | - | - | todo |

## M4 – AI agent with tools and MCP server

| ID | Task | Spec | Branch | PR | Status |
|----|------|------|--------|----|--------|
| T-011 | AI agent with tools and MCP server | - | - | - | todo |

## M5 – Load testing and observability

| ID | Task | Spec | Branch | PR | Status |
|----|------|------|--------|----|--------|
| T-012 | Load testing and observability | - | - | - | todo |
| T-014 | Scale and portability (P0 portability, P1 shard-ready) | [scale](specs/platform/003-scale-and-portability.md) | feat/003-scale-and-portability | - | in progress |

## Chore

| ID | Task | Spec | Branch | PR | Status |
|----|------|------|--------|----|--------|
| T-013 | Tasks milestones restructure | - | chore/tasks-milestones | [#6](https://github.com/GodwinJacobR/mock-bank/pull/6) | in progress |
