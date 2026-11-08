# System flow

Source: [specs/000-architecture.md](../specs/000-architecture.md). Note: there is no separate "transfers" service. Transfers live in the **ledger**.

## Components

```mermaid
flowchart LR
  UI["web UI (htmx + askama)"] --> GW[gateway]
  GW -- sync HTTP --> ACC[accounts]
  GW -- sync HTTP --> LED["ledger (transfers, balances)"]
  GW --> R[("Redis: sessions, rate limit, idem, cache")]
  ACC --> PG[("Postgres, schema per service")]
  LED --> PG
  ACC -- outbox --> K{{Kafka}}
  LED -- outbox --> K
  K -- accounts.events.v1 --> LED
  K -- ledger.transfers.v1 --> AST["assistant (phase 3)"]
```

- Only the gateway calls accounts and ledger (sync). Services never call each other over HTTP.
- State moves between services via Kafka (ledger builds `account_projection` from `accounts.events.v1`).
- Redis is phase 2; assistant and MCP are phase 3-4 (see spec phases).

## Transfer sequence

```mermaid
sequenceDiagram
  participant UI as web UI
  participant GW as gateway
  participant R as Redis
  participant L as ledger
  participant DB as Postgres
  participant K as Kafka
  UI->>GW: POST /transfers (Idempotency-Key)
  GW->>GW: auth, rate limit
  GW->>R: idem:{key} (phase 2)
  GW->>L: POST /transfers
  L->>DB: BEGIN
  L->>DB: check accounts, idempotency key
  L->>DB: lock balances (id order)
  L->>DB: funds check, insert transfer + 2 entries, update balances, idem record, outbox row
  L->>DB: COMMIT
  L-->>GW: 201
  GW-->>UI: 201
  L->>K: outbox relay publishes TransferPosted
```

## Account page

Gateway calls accounts (list) and ledger (balances), then composes the page. Redis balance cache sits in front of the ledger call (phase 2).

## Guarantees

- Double entry: entries sum to 0 per transfer.
- Idempotent writes (`Idempotency-Key`).
- Events at-least-once; consumers dedup on `event_id`.
