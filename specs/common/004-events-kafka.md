# Events (Kafka)

Status: draft

## Goal
Library crate `common/events` (`bank-events`): one reusable way to publish and consume Kafka events, used by every service (directly by consumers, and by `common/005-outbox` for publishing).

## Scope (in)
- Event envelope type and typed topic names.
- `Publisher` with safe producer settings.
- `Consumer` runner: manual commit, dedup, retry, DLQ, graceful shutdown, replay from earliest.
- Trace context propagation in Kafka headers.

## Out of scope
- The outbox table and relay (`common/005-outbox`).
- Avro/Protobuf and schema registry (plain JSON only).
- Topic creation policy beyond a dev helper (Compose creates topics: 3 partitions, `accounts.events.v1` infinite retention).
- Log compaction.

## Design
**Client crate**: `rdkafka` (librdkafka binding, tokio `FutureProducer` and `StreamConsumer`). Chosen for idempotent producer, manual commits and mature behaviour; the crate wraps it so services never import `rdkafka`. Pure-Rust clients are rejected for weaker idempotence/transaction support.

**Envelope** (JSON, versioned by topic name and `schema_version`)
```rust
pub struct EventEnvelope<T> {
    pub event_id: Uuid,
    pub r#type: String,          // e.g. "TransferPosted"
    pub occurred_at: DateTime<Utc>, // RFC3339
    pub schema_version: u32,
    pub aggregate_id: String,
    pub payload: T,
}
```

**Typed topics**
```rust
pub struct Topic<T> { pub name: &'static str, _p: PhantomData<T> }
pub const ACCOUNTS_EVENTS: Topic<AccountEvent> = Topic::new("accounts.events.v1");
pub const LEDGER_TRANSFERS: Topic<TransferEvent> = Topic::new("ledger.transfers.v1");
impl<T> Topic<T> { pub fn dlq(&self) -> String /* "<topic>.dlq.v1" derived from the base name */ }
```
- DLQ naming: `<topic>.dlq.v1`, e.g. `accounts.events.dlq.v1` (the `.v1` suffix of the source topic is replaced). Event payload structs (`AccountEvent`, `TransferEvent`) live in this crate so producers and consumers share them.

**Publisher**
- Settings: `acks=all`, `enable.idempotence=true`, `max.in.flight.requests.per.connection=5`, `compression.type=lz4`, bounded `message.timeout.ms`.
- Adds headers: `traceparent` (from the current span via `common/003-telemetry`), `event_id`, `content-type: application/json`.
- Key is mandatory; per-key ordering relies on it.
```rust
#[async_trait]
pub trait Publisher: Send + Sync {
    async fn publish(&self, topic: &str, key: &str, headers: Headers, payload: &[u8]) -> Result<(), PublishError>;
}
impl KafkaPublisher { pub fn new(cfg: KafkaConfig) -> Result<Self>; }
// typed convenience:
async fn publish_event<T: Serialize>(p: &dyn Publisher, t: &Topic<T>, key: &str, ev: &EventEnvelope<T>) -> Result<()>;
```
`publish` resolves only after the broker acks (all in-sync replicas).

**Consumer runner**
```rust
#[async_trait]
pub trait Handler<T>: Send + Sync {
    async fn handle(&self, tx: &mut sqlx::Transaction<'_, Postgres>, ev: EventEnvelope<T>) -> Result<(), HandlerError>;
}
pub trait DedupStore: Send + Sync { /* default impl: SqlxDedup over `processed_events` */ }
pub struct ConsumerConfig {
    pub group_id: String, pub topic: String, pub max_retries: u32 /*5*/,
    pub backoff: Backoff /*exp 200ms..30s, jitter*/, pub start: StartFrom /*Committed | Earliest*/,
}
pub async fn run<T: DeserializeOwned>(cfg: ConsumerConfig, pool: PgPool, h: impl Handler<T>, shutdown: CancellationToken) -> Result<()>;
```
Per message:
1. Deserialize. Failure (poison) goes straight to DLQ.
2. Begin a DB transaction. `INSERT INTO processed_events(event_id, consumer, processed_at) ... ON CONFLICT DO NOTHING`; if no row was inserted it is a duplicate: commit nothing, commit the Kafka offset, continue.
3. Call `handler.handle(tx, event)` in the same transaction, then commit the DB transaction, then commit the Kafka offset (manual, `enable.auto.commit=false`). Dedup record and side effects are atomic, so a crash between DB commit and offset commit replays the event harmlessly.
4. Handler error: roll back, retry with exponential backoff and jitter up to `max_retries` (partition is paused in place to preserve per-key order). After exhaustion publish the original bytes plus headers `dlq-reason`, `dlq-attempts`, `dlq-source-topic/partition/offset` to the DLQ topic, then commit the offset.
5. Metrics via `metrics`: `events_consumed_total`, `events_dlq_total`, `events_duplicate_total`, `consumer_lag`.

**`processed_events` table** (provided by each consuming service in its schema, see `ledger/002-schema`):
`processed_events(event_id uuid, consumer text, processed_at timestamptz, primary key (event_id, consumer))`. The crate takes the schema-qualified table name via config.

**Graceful shutdown**: on cancel, stop polling, finish the in-flight message (handler + both commits), close the consumer (leave group), return. Hard deadline configurable (default 30 s).

**Replay from earliest**: `StartFrom::Earliest` (CLI/env flag `REPLAY_FROM_EARLIEST=true`) uses a fresh group id suffix or seeks to beginning on assignment, ignoring committed offsets. The service is responsible for truncating its projection and `processed_events` rows for that consumer first (documented helper `reset_consumer(pool, consumer)`).

## Dependencies
- `common/001-money-and-errors`, `common/003-telemetry`.

## Acceptance criteria
- [ ] Publisher uses `acks=all` and idempotence (asserted from config in a unit test).
- [ ] Published messages carry a key and `traceparent`.
- [ ] Offset is committed only after the handler transaction commits.
- [ ] Redelivered event (same `event_id`) invokes the handler once.
- [ ] Failing handler is retried with backoff, then lands on `<topic>.dlq.v1` with reason headers; the main partition continues.
- [ ] Malformed JSON goes to the DLQ without retries.
- [ ] Shutdown signal finishes the in-flight message and exits without losing or duplicating effects.
- [ ] `StartFrom::Earliest` re-reads all messages from offset 0.
- [ ] Per-key ordering is preserved through retries.

## Test plan
- Unit: backoff schedule, DLQ topic naming, header encoding, envelope serde.
- Integration (testcontainers Kafka + Postgres):
  - publish N events, consume, assert handler effects once each;
  - publish duplicates, assert single effect;
  - kill the consumer after handler commit but before offset commit (fault hook), restart, assert no double effect;
  - handler that always fails: assert DLQ message and headers, main topic progresses;
  - cancel mid-stream, assert clean exit;
  - earliest replay after resetting state rebuilds the same result.
