# Outbox

Status: draft

## Goal
Library crate `common/outbox` (`bank-outbox`): transactional outbox so a service can change its data and emit an event atomically, with a reusable relay that publishes to Kafka. Used by accounts and ledger.

## Scope (in)
- Standard `outbox` table DDL that each service schema includes.
- `enqueue` inside the caller's sqlx transaction.
- Relay task: poll, publish through `common/events`, mark published.
- Metrics hooks.

## Out of scope
- Consuming events (`common/004-events-kafka`).
- CDC / Debezium.
- Deleting old published rows beyond an optional cleanup helper.

## Design
**Standard DDL** (copied into each service's init migration; schema-qualified there)
```sql
CREATE TABLE outbox (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  topic        text        NOT NULL,
  key          text        NOT NULL,
  payload      jsonb       NOT NULL,       -- full EventEnvelope
  created_at   timestamptz NOT NULL DEFAULT now(),
  published_at timestamptz,
  attempts     integer     NOT NULL DEFAULT 0,
  last_error   text
);
CREATE INDEX outbox_unpublished_idx ON outbox (created_at) WHERE published_at IS NULL;
```
`id` equals the envelope `event_id`, so consumers can dedup on it.

**API sketch**
```rust
pub async fn enqueue<T: Serialize>(
    tx: &mut sqlx::Transaction<'_, Postgres>,
    topic: &Topic<T>, key: &str, event: &EventEnvelope<T>,
) -> Result<(), OutboxError>;   // INSERT only; never touches Kafka

pub struct RelayConfig {
    pub table: String /* "ledger.outbox" */, pub batch_size: i64 /*100*/,
    pub poll_interval: Duration /*200ms idle*/, pub backoff: Backoff /*on publish error: 200ms..30s*/,
}
pub trait RelayMetrics { fn published(&self, n: usize); fn failed(&self); fn lag(&self, oldest_unpublished: Duration); }
pub async fn run_relay(cfg: RelayConfig, pool: PgPool, publisher: Arc<dyn Publisher>,
                       metrics: impl RelayMetrics, shutdown: CancellationToken) -> Result<()>;
```

**Relay loop**
1. Begin transaction. `SELECT id, topic, key, payload FROM <table> WHERE published_at IS NULL ORDER BY created_at, id LIMIT $batch FOR UPDATE SKIP LOCKED`.
2. Publish rows in order via `Publisher::publish` (acks=all). Rows with the same key are published sequentially to keep per-key order; a failure stops the batch at that row.
3. `UPDATE ... SET published_at = now()` for successfully published ids (and `attempts`, `last_error` for the failed one), commit.
4. Empty batch: sleep `poll_interval`. Publish error: exponential backoff. Report lag (age of oldest unpublished row) every loop.
Multiple relay instances are safe: `SKIP LOCKED` partitions the work (ordering across instances is then per batch only; run one relay per service locally to keep global order per key).

**Crash safety and duplicates**
- Atomicity: the event row commits with the business change, or neither does.
- Crash before commit of step 3 (after publishing) leaves rows unpublished, so they are published again: at-least-once, duplicates possible. Consumers dedup on `event_id`.
- No event is lost: a row is only marked published after the broker acked.
- A relay killed while holding row locks releases them when its connection drops.

## Dependencies
- `common/004-events-kafka`.

## Acceptance criteria
- [ ] `enqueue` inside a rolled-back transaction leaves no outbox row.
- [ ] Every committed outbox row is eventually published with the stored topic and key.
- [ ] Two relays running concurrently never publish the same row in the same moment (SKIP LOCKED) and together publish all rows.
- [ ] Killing the relay between publish and mark causes duplicates, never loss.
- [ ] Publish failures back off and the row is retried; `attempts` and `last_error` are recorded.
- [ ] Lag and published-count metric hooks are called.
- [ ] Per-key order is preserved in the topic.

## Test plan
- Integration (testcontainers Postgres + Kafka):
  - enqueue in committed and rolled-back transactions;
  - publish 1000 rows, assert all arrive with keys;
  - kill the relay mid-run (abort the task after publish, before the mark; fault hook), restart, assert all rows arrive and duplicates share `event_id`;
  - run two relays concurrently, assert all rows arrive;
  - stop Kafka briefly, assert backoff and later delivery.
- Unit: backoff schedule, batch selection ordering.
