# Saga library

Status: draft

## Goal
Rust library crate `crates/saga` (`bank-saga`): a reusable saga engine that runs inside whichever service links it. A host service defines a saga type as an ordered list of steps; each step runs arbitrary async host code (an HTTP call, a command sent over Kafka, anything). The library persists the saga in the host's own Postgres database and handles retries, timeouts, compensation (undo in reverse) and crash recovery. It is not a service and has no process of its own. First host: transfers (`transfers/001-service`).

## Scope (in)
- Crate layout, cargo features, public API (`migrate`, `start`, `resume`, saga definitions, `Step`, `Runner`, `Config`).
- Library-owned tables (`sagas`, `saga_steps`, and `saga_outbox` with the `outbox` feature) and their embedded migrations.
- Runner (claim, run, record result), retries and backoff, timeouts, compensation, sweeper, late-result rule.
- Idempotency key helper.
- Optional outbox + relay module (feature `outbox`) for steps that send commands.
- `saga_id` on logs and trace spans, metrics hooks.

## Out of scope
- Any business logic or transport: the library knows nothing about accounts, ledger, debit/credit, HTTP or Kafka topics (the optional outbox module only moves opaque `topic`/`key`/`payload` rows).
- A central orchestrator service or its own database.
- Cross-database distributed transactions (2PC): saga tables live in the host's database.
- Databases other than Postgres (`sqlx` only).
- Multi-saga parent/child chains.

## Design

### Crate layout and features
```
crates/saga/
  migrations/            embedded SQL (sqlx::migrate!), templated on schema/prefix
  src/lib.rs             re-exports, SagaError
  src/config.rs          Config
  src/definition.rs      SagaDefinition, SagaRegistry, validation
  src/step.rs            Step trait, Outcome, Next, Failure, StepFailure, HookCtx
  src/engine.rs          Saga (start, resume), state transitions
  src/runner.rs          Runner (claim loop + sweeper)
  src/keys.rs            idempotency_key, undo_key
  src/outbox/            feature `outbox`: enqueue, Relay, Publisher trait, ReplyHandler
```
| Feature | Adds |
|---|---|
| (default) | everything except the outbox module |
| `outbox` | `saga_outbox` table + migration, `outbox::enqueue`, `Relay`, `Publisher` trait, `outbox::ReplyHandler` |
| `kafka` (implies `outbox`) | `rdkafka` `Publisher` and reply-topic consumer adapter |

Dependencies: `sqlx` (postgres, uuid, json, chrono), `tokio`, `serde`/`serde_json`, `uuid` (v5), `thiserror`, `tracing`, `async-trait`. Following the workspace Rust rules: library errors with `thiserror`, no `unwrap`/`expect` outside tests, `cargo fmt` and `cargo clippy -- -D warnings` clean. `anyhow` is only used in host binaries.

### Public API
```rust
// config
pub struct Config {
    pub schema: Option<String>,        // e.g. Some("transfers"); None = connection search_path
    pub table_prefix: String,          // default "" ; "pay_" -> pay_sagas, pay_saga_steps, pay_saga_outbox
    pub max_attempts: u32,             // default 5
    pub step_timeout: Duration,        // default 30 s
    pub stale_after: Duration,         // default 60 s
    pub sweep_interval: Duration,      // default 15 s
    pub poll_interval: Duration,       // default 500 ms
    pub undo_max_backoff: Duration,    // default 30 s
    pub base_backoff: Duration,        // default 500 ms
    pub max_backoff: Duration,         // default 10 s
    pub batch_size: u32,               // sagas claimed per poll, default 16
}

// migrations: creates/updates the library tables in the HOST's database
pub async fn migrate(pool: &PgPool, config: &Config) -> Result<(), SagaError>;

// definitions
pub struct SagaDefinition { /* type name + ordered steps */ }
impl SagaDefinition {
    pub fn builder(saga_type: &str) -> SagaDefinitionBuilder;      // .step(impl Step).retry(...).build()?
}
pub struct SagaRegistry { /* type name -> SagaDefinition */ }

// engine, one per host process
pub struct Saga { /* pool, Config, SagaRegistry */ }
impl Saga {
    pub fn new(pool: PgPool, config: Config, registry: SagaRegistry) -> Self;
    /// Inserts the sagas row (RUNNING, current_step = first step, deadline = now()) in the CALLER's transaction.
    pub async fn start(&self, tx: &mut Transaction<'_, Postgres>, saga_type: &str, context: Value) -> Result<Uuid, SagaError>;
    /// Same, with a caller-chosen saga_id (e.g. the host's business id).
    pub async fn start_with_id(&self, tx: &mut Transaction<'_, Postgres>, saga_id: Uuid, saga_type: &str, context: Value) -> Result<(), SagaError>;
    /// Delivers the result of a `Wait` step. Applied or dropped per the late-result rule.
    pub async fn resume(&self, saga_id: Uuid, step: &str, result: StepResult) -> Result<Resumed, SagaError>;
    pub fn runner(&self) -> Runner;
}
pub enum StepResult { Ok, Failed { code: String } }
pub enum Resumed { Applied, Dropped }

// idempotency keys (UUIDv5 helper)
pub fn idempotency_key(saga_id: Uuid, step: &str) -> Uuid;        // UUIDv5(namespace = saga_id, name = step)
pub fn undo_key(saga_id: Uuid, step: &str) -> Uuid;               // UUIDv5(namespace = saga_id, name = "{step}:undo")

// runner: the host spawns it
impl Runner { pub async fn run(self, shutdown: impl Future<Output = ()>) -> Result<(), SagaError>; }

// feature `outbox`
pub mod outbox {
    pub async fn enqueue(exec: impl PgExecutor, config: &Config, topic: &str, key: &str, payload: Value) -> Result<(), SagaError>;
    pub trait Publisher { async fn publish(&self, topic: &str, key: &str, payload: &Value) -> Result<(), PublishError>; }
    pub struct Relay { /* polls saga_outbox, publishes, sets published_at */ }
    pub struct ReplyHandler { /* maps a (saga_id, step, result) message to Saga::resume */ }
}
```
Errors are `SagaError` (`thiserror`): `Db(sqlx::Error)`, `Migrate(sqlx::migrate::MigrateError)`, `InvalidDefinition(String)`, `UnknownType(String)`, `UnknownStep(String)`, `NotFound(Uuid)`, `Serde(serde_json::Error)`, `Publish(PublishError)`. Step code returns `StepFailure { code: String, message: String }` (also `thiserror`).

### Step contract
A saga type is an ordered list of steps. Each step implements:
```rust
#[async_trait]
pub trait Step: Send + Sync {
    fn name(&self) -> &str;                                   // unique within the type, no ':' ("{name}:undo" is reserved)
    fn has_compensation(&self) -> bool { false }              // false = non-undoable
    async fn run(&self, ctx: &StepCtx) -> Result<Outcome, StepFailure>;
    async fn compensate(&self, ctx: &StepCtx) -> Result<Outcome, StepFailure> { Ok(Outcome::Done) }
    async fn on_success(&self, hook: &mut HookCtx<'_>) -> Result<Next, SagaError> { Ok(Next::InOrder) }
    async fn on_failure(&self, hook: &mut HookCtx<'_>, code: &str, attempt: u32) -> Result<Failure, SagaError> { Ok(Failure::Compensate) }
}
pub enum Outcome { Done, Wait }
pub enum Next    { InOrder, Step(String), Done, Wait }
pub enum Failure { Retry, Compensate, Skip, Fail }
```
- `StepCtx` gives `saga_id`, `step`, `attempt`, a read-only `context`, and the keys (`idempotency_key`, `undo_key`). `run` is arbitrary async host code. It returns `Done` (finished, effect applied), `Wait` (the effect was requested; the result arrives later through `resume`), or `Err(StepFailure)` (it failed with a business or transient `code`). The library applies `step_timeout` around `run` and treats a timeout as code `timeout`.
- `run` and `compensate` must be idempotent: the library re-runs them on retry, timeout and crash recovery with the same key (the key does not depend on `attempt`). They run outside any library transaction (no lock is held during the call).
- `on_success` / `on_failure` are the decision hooks. They run inside the result transaction (below) with `HookCtx { tx, saga_id, context: &mut Value, step }`, so a host can mirror status into its own tables or enqueue its own outbox rows atomically with the saga update. They make no external calls.
  - `on_success` stores what it needs in `context` and returns `Next`: `InOrder` (next step; after the last step, `COMPLETED`), `Step(name)` (any step of the type, looked up by name, for branching), `Done` (`COMPLETED`), `Wait` (park, see Waiting and resume).
  - `on_failure` returns `Failure`: `Retry` (same key, bounded by `max_attempts`; past the limit it counts as `Compensate`), `Compensate` (set `failure_reason = code`, start the undo), `Skip` (continue as if done), `Fail` (status `FAILED`).
- `compensate` undoes a step that already succeeded, in reverse order, under the step name `{name}:undo`.
- Registration rules (checked by `SagaDefinitionBuilder::build`, error `InvalidDefinition`): names unique per type, no `:`, non-empty, and no undoable step after a non-undoable one. Non-undoable steps go last. The first one may still be rejected (the earlier steps are then undone); once it has succeeded it is the point of no return and the saga only moves forward. Later non-undoable steps must not fail on business grounds; their only failures are transient (`timeout`) and retried until they succeed. A timeout on the first non-undoable step is also retried without limit (the outcome is unknown and it may have applied).
- Retry policy per type or per step: `max_attempts`, `base_backoff`, `max_backoff` (exponential, with jitter); undo steps default to unlimited attempts (cap `undo_max_backoff`) with an alert.

### Idempotency keys
`idempotency_key(saga_id, step)` = `UUIDv5(namespace = saga_id, name = step)`; `undo_key(saga_id, step)` = `UUIDv5(namespace = saga_id, name = "{step}:undo")`. A pure function of `(saga_id, step)`, so every re-run (retry, sweeper, crash recovery) carries the same key. The library supplies the key; honouring it (dedupe) is the receiving side's job.

### Tables
Created by `saga::migrate(&pool, &config)` in the host's own database, in `config.schema` (if set) with every table name prefixed by `config.table_prefix`, so several services can each embed their own copy and one service can host several sets. The DDL below uses the default names (no prefix).
```sql
CREATE TABLE sagas (
  saga_id        uuid        PRIMARY KEY,
  type           text        NOT NULL,                 -- which step list
  status         text        NOT NULL DEFAULT 'RUNNING',
  current_step   text        NOT NULL,                 -- step name; RUNNING: current step (`debit`); COMPENSATING: step being undone (`debit:undo`)
  context        jsonb       NOT NULL,                 -- input and values kept by hooks
  failure_reason text,                                 -- set when compensation starts
  deadline       timestamptz,                          -- when the step is due or overdue; NULL = not scheduled
  updated_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT sagas_status_chk CHECK (status IN ('RUNNING','COMPENSATING','COMPLETED','REVERSED','FAILED'))
);
CREATE INDEX sagas_active_idx ON sagas (updated_at) WHERE status IN ('RUNNING','COMPENSATING');  -- sweeper
CREATE INDEX sagas_deadline_idx ON sagas (deadline) WHERE deadline IS NOT NULL;                 -- claim and sweeper

CREATE TABLE saga_steps (                                 -- append-only
  saga_id uuid        NOT NULL,
  step    text        NOT NULL,                        -- action name, or "{name}:undo" for an undo
  status  text        NOT NULL,
  ts      timestamptz NOT NULL DEFAULT now(),
  payload jsonb,                                       -- attempt number, result code, error text
  CONSTRAINT saga_steps_status_chk CHECK (status IN ('started','done','failed','compensated'))
);
CREATE INDEX saga_steps_saga_idx ON saga_steps (saga_id, ts);

-- feature `outbox` only
CREATE TABLE saga_outbox (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  topic        text        NOT NULL,
  key          text        NOT NULL,
  payload      jsonb       NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now(),
  published_at timestamptz,
  attempts     integer     NOT NULL DEFAULT 0,
  last_error   text
);
CREATE INDEX saga_outbox_unpublished_idx ON saga_outbox (created_at) WHERE published_at IS NULL;
```
- `saga_steps` rows are only inserted, never updated or deleted (the host's runtime role is granted `SELECT, INSERT` only). `started` is written when the runner claims a step (payload: `attempt`), then `done` or `failed` (payload: code) for a forward step; an undo writes `started` and `compensated` under `{step}:undo`. Each row is written in the same transaction as the `sagas` update it belongs to.
- History of one saga: `SELECT * FROM saga_steps WHERE saga_id = $1 ORDER BY ts;`. `sagas` is the current-state row; `saga_steps` is the audit trail.
- Migrations are embedded (`sqlx::migrate!`), versioned and recorded in a per-set migrations table (`{prefix}saga_migrations`), so `migrate` is idempotent and safe to call on every start. The host's DB owner role runs it (the runtime role needs no DDL).
- Statuses: `RUNNING` (forward steps), `COMPENSATING` (undo in reverse), `COMPLETED`, `REVERSED` (failed and the done steps were undone), `FAILED` (the first step failed with nothing to undo, or `on_failure` returned `Fail`). Final rows are immutable.
- Hosts keep business fields in their own tables and link by `saga_id`.

### Start
`start` / `start_with_id` run in the caller's open transaction: insert the `sagas` row (`RUNNING`, `current_step` = first step name, `context`, `deadline = now()`). The host's business writes (e.g. its own row) are in the same transaction, so they commit or roll back together. Nothing is executed in `start`; the runner picks the saga up after commit.

### Runner
The host spawns `saga.runner().run(shutdown)` as a background task (`tokio::spawn`); it runs the claim loop and the sweeper. Any number of pods may run it: all claims use `FOR UPDATE SKIP LOCKED`.

Each cycle (every `poll_interval`; the sweeper query every `sweep_interval`):
1. **Claim** (short transaction): `SELECT ... FROM sagas WHERE status IN ('RUNNING','COMPENSATING') AND deadline <= now() ORDER BY deadline LIMIT $batch FOR UPDATE SKIP LOCKED`. For each saga, by the latest `saga_steps` row of `current_step`:
   - not `started` (new step, or a retry whose backoff is over): append `started` (attempt = previous + 1), set `deadline = now() + step_timeout` (the lease), commit, then run the step.
   - `started` and `deadline` passed (no result: the call hung, the pod died, or a `Wait` result never arrived): treat as a failure with code `timeout` and go to step 3 (no re-run yet).
2. **Run** (no lock held): `run` for `RUNNING`, `compensate` for `COMPENSATING`, under `step_timeout`.
3. **Record the result** (one transaction): lock the row `FOR UPDATE`, apply the late-result rule, then:
   - `Ok(Done)` or a resumed `Ok`: append `done` (or `compensated`), call `on_success`, move `current_step` per `Next`, set `deadline = now()` for the next step, update `context`/`status`/`updated_at`. Last step done -> `COMPLETED`; last undo done -> `REVERSED`.
   - `Ok(Wait)`: leave the `started` row, set `deadline = now() + step_timeout` (the wait timeout), commit. The result arrives via `resume`.
   - `Err(StepFailure)` or timeout: append `failed` (payload code), call `on_failure(code, attempt)`, then act on `Failure`: `Retry` (set `deadline = now() + backoff`; the next claim appends `started` attempt + 1; past `max_attempts` it counts as `Compensate`), `Compensate` (set `failure_reason`, status `COMPENSATING`, `current_step` = `{latest completed undoable step}:undo`; if the failure was a timeout on an undoable step, that step is undone too since its outcome is unknown; with nothing done yet the status is `FAILED`), `Skip`, `Fail`.
   - The hook writes (host tables, host outbox rows) are in this same transaction; a hook error rolls everything back and the step is claimed again after its lease.
4. **Sweeper**: the claim query already covers every active saga that has a `deadline`. In addition, every `sweep_interval` it picks `status IN ('RUNNING','COMPENSATING') AND deadline IS NULL AND updated_at < now() - stale_after` (an interrupted write) and sets `deadline = now()` so the next claim runs it.

Rules
- Undo steps run in reverse order of the completed steps. Undo failures are retried until they succeed (backoff capped at `undo_max_backoff`) and raise an alert (`saga_undo_stuck`); compensation is never abandoned.
- A business failure in a non-undoable step after the point of no return is a bug: the library alerts and stops the saga for manual action.
- A `Failure::Retry` re-runs the same step with the same key; a retry never guesses the outcome of a timed-out call.

### Waiting and resume
`Outcome::Wait` (from `run`) or `Next::Wait` (from `on_success`) parks the saga with a `deadline`. It is resumed by `Saga::resume(saga_id, step, StepResult)`, called by host code (an HTTP callback, a Kafka consumer, a timer) or, with the `outbox` feature, by `outbox::ReplyHandler`. `resume` runs step 3 of the runner (lock, late-result rule, `on_success` / `on_failure`). Hosts that never use `Wait` never call `resume`.

### Late-result rule
A result (from a run, a resume, or a timeout) is applied only if the saga is not final, `step` equals `current_step` (the step name for `RUNNING`, its `{name}:undo` for `COMPENSATING`), that step has no `done`/`compensated` row yet, and, for a run result, its `attempt` matches the latest `started` row. Otherwise it is dropped and nothing is written (`Resumed::Dropped`). So a result arriving after a timeout, a retry or compensation never changes the saga. Because a timeout re-runs the step with the same key rather than assuming failure, the real outcome is learned from the receiving side's stored result for that key. A receiver that can see both a forward key and its `{step}:undo` key must reject a forward request whose undo it has already processed (so a late action cannot land after its undo), and must answer an undo for an action it never applied as ok with no effect. This is a receiver contract, documented here because the library's retry and timeout semantics rely on it.

### Optional outbox and relay (feature `outbox`)
For steps that send commands to another service asynchronously. A step's `run` calls `outbox::enqueue(...)` to insert a `saga_outbox` row (opaque `topic`, `key`, `payload`), then returns `Outcome::Wait`. `Relay` (spawned by the host like the runner) polls unsent rows (`published_at IS NULL`, `FOR UPDATE SKIP LOCKED`), calls the host-supplied `Publisher`, and sets `published_at`; it is at-least-once, so receivers dedupe by idempotency key. The Kafka `Publisher` (feature `kafka`) keys messages by the given `key`. `ReplyHandler` takes a parsed reply `(saga_id, step, result)` from the host's consumer and calls `resume`; the host commits the consumer offset after `resume` returns. The library does not define topics or message shapes; the host does. Steps that call a service directly (HTTP) do not use any of this. A host's own event outbox (`common/005-outbox`) is separate and unaffected.

### Crash recovery
State changes only in the start transaction (host-owned), the claim transaction and the result transaction. A run or relay publish happens between transactions and is repeated safely by key.

| Crash point | State on restart | Result |
|---|---|---|
| Before the host's start tx commits | nothing stored | the host's client retries with its own idempotency key |
| After start, before any claim | `deadline <= now()` | the runner claims it |
| After claim, before `run` returns | `started` row, lease running | at the lease deadline the next claim sees `started` + overdue -> `timeout` -> `on_failure` (retry re-runs with the same key) |
| After `run` finished its effect, before the result tx | `started` row | same as above; the receiver dedupes by key |
| During the result tx | rolled back | as above |
| After the result tx | next step `deadline = now()` | claimed normally |
| `Wait` parked, result lost | `started` + overdue | timeout -> `on_failure`; retry re-runs `run` with the same key |
| Outbox row written, relay dies before `published_at` | unsent | relay re-publishes; receivers dedupe |
| Two pods at once | `SKIP LOCKED` / row lock | one claims; a racing result tx serializes on `FOR UPDATE` and drops as already applied |

### Observability
Every log line and trace span written while handling a saga carries `saga_id` (plus `type` and `step`), via a `tracing` span opened per claim, per result and per resume (`common/003-telemetry`).
Metrics (labels type/step): step latency and attempts, sagas by final status, running and compensating counts, oldest running age, timeouts, retries, sweeper re-schedules, `saga_undo_stuck` gauge, and with `outbox` the unsent row count and relay publish errors.

## Worked example (generic, two steps)
A host defines type `order` with steps `reserve` (undoable, direct HTTP call) and `notify` (non-undoable). `saga_id = 6f1c2a40-0000-4000-8000-000000000001`.
1. Host tx: its own row + `start_with_id(tx, saga_id, "order", ctx)`; commit. `sagas`: `RUNNING`, `current_step = reserve`, `deadline = now()`.
2. Runner claims: `saga_steps` `reserve started` (attempt 1); `run` calls the service with `idempotency_key(saga_id, "reserve")` and returns `Done`.
3. Result tx: `reserve done`, `on_success` -> `InOrder`; `current_step = notify`.
4. Next claim: `notify started`, `run` -> `Done`; result tx: `notify done`, status `COMPLETED`.
If `notify` had failed with a business error, `reserve:undo` would run `compensate` with `undo_key(saga_id, "reserve")` and the saga ends `REVERSED`. The transfer saga (debit, credit, Kafka commands) is defined in `transfers/001-service`.

## Dependencies
- `common/003-telemetry`, `platform/002-workspace-and-compose`. With feature `outbox`: `common/005-outbox` (same pattern; the library ships its own table) and, with `kafka`, `common/004-events-kafka`.

## Acceptance criteria
- [ ] `migrate` creates `sagas`, `saga_steps` (and `saga_outbox` with the `outbox` feature) in the configured schema with the configured prefix; it is idempotent; two sets with different prefixes coexist in one database.
- [ ] `start`/`start_with_id` insert the `sagas` row in the caller's transaction: a rollback of the caller's transaction leaves no saga row.
- [ ] A saga with N steps runs them in order; `current_step`, `deadline` and `updated_at` advance in the same transaction as the step result and the hook's writes.
- [ ] A step's `run` can be any async code (HTTP, send a command, nothing); the library has no dependency on a transport or on accounts/ledger concepts.
- [ ] `idempotency_key` / `undo_key` return `UUIDv5(saga_id, step)` / `UUIDv5(saga_id, "{step}:undo")` and are identical on every re-run.
- [ ] A failed step with the default `on_failure` sets `COMPENSATING`, runs `compensate` in reverse, marks each `compensated`, and ends `REVERSED`; a first-step failure ends `FAILED`.
- [ ] Undo failures are retried until success and raise `saga_undo_stuck`.
- [ ] `Retry` re-runs with the same key and backoff up to `max_attempts`, then compensates; `Skip` continues; `Fail` ends `FAILED`.
- [ ] The last step done ends `COMPLETED`; `Next::Step(name)` jumps to that step.
- [ ] Registering a type with duplicate or `:`-containing names, or an undoable step after a non-undoable one, is rejected with `InvalidDefinition`.
- [ ] A `Wait` step parks with a `deadline`; `resume` completes it via `on_success`/`on_failure`; a result for a non-current or already-done step, or a stale attempt, is dropped without writing.
- [ ] A saga past its `deadline` with a `started` row is handled as `timeout` per `on_failure`; a saga with `deadline IS NULL` and stale `updated_at` is re-scheduled; final sagas are ignored.
- [ ] Two pods running `Runner` never run or apply the same step concurrently (`SKIP LOCKED`, row lock).
- [ ] Each row of the crash table ends in the same final state with each effect applied once (given an idempotent step).
- [ ] `saga_steps` is append-only and the history query returns every step in order, including `{step}:undo` `compensated` rows.
- [ ] A hook error rolls back the result transaction (no `saga_steps`/`sagas` change).
- [ ] With `outbox`: `enqueue` + `Relay` publish each row at least once and set `published_at`; unsent rows are those with `published_at IS NULL`; `ReplyHandler` calls `resume`.
- [ ] The same `saga_id` appears in every log line and span for a saga.
- [ ] The crate builds with and without the `outbox`/`kafka` features and passes `cargo fmt` and `cargo clippy -- -D warnings`; no `unwrap`/`expect` outside tests.

## Test plan
- Unit: definition validation (unique names, no `:`, undo ordering), key derivation (UUIDv5 vectors), backoff schedule, late-result rule table, `current_step` lookup by name incl. `:undo`, `Next`/`Failure` transitions.
- Integration (testcontainers Postgres; Kafka only for the `kafka` feature; a fake step with scriptable results): happy path, first-step failure (`FAILED`), later-step failure (`REVERSED`), undo failing then succeeding, `Retry`/`Skip`/`Fail`/`Wait`+`resume`/branching, timeout and retry with the same key, killing the runner mid-run and mid-result, duplicate and late results, two runners on one database (no double run), sweeper path, start rollback, hook error rollback, two prefixes in one database, `saga_steps` history and append-only grant, outbox relay restart and duplicate publish.
