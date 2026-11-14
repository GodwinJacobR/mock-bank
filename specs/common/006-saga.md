# Saga runner

Status: draft

## Goal
Library crate `common/saga` (`bank-saga`): a reusable runner for multi-step workflows across services. A service defines a saga type as a step list; the runner executes it with idempotent steps, persisted state, retries, undo in reverse, and crash recovery. First user: transfers (`transfers/001-service`).

## Scope (in)
- Step definition, generic `sagas` state table DDL, the runner loop, failure handling, crash recovery (sweeper, row lock).
- Integration with Kafka consumers (`common/004-events-kafka`) and the outbox (`common/005-outbox`).
- Metrics hooks.
- Step history table (`saga_steps`) and `saga_id` on logs and trace spans.

## Out of scope
- Business logic of any step (owned by the service defining the saga type).
- Cross-service distributed transactions (2PC), saga orchestration across several services' databases: the state table lives in the owning service's schema.
- Multi-saga parent/child chains.

## Design

**Step list**. A saga type is an ordered list of steps.

| Field | Meaning |
|---|---|
| `name` | step name, unique within the type |
| `action` | async fn(ctx) -> Ok / Transient(err) / Permanent(err); must be idempotent |
| `undo` | async fn(ctx) -> same result type, or `none` |
| idempotency key | `{saga_id}-{step}` for the action; `{saga_id}-{step}-undo` for the undo. Passed to the callee in `ctx` |
| retry policy | `max_attempts`, `base_backoff`, `max_backoff`, jitter; undo steps default to unlimited attempts (cap 30 s) with an alert |

Rules
- Every action and undo is idempotent: same key, same effect (the callee returns the stored response on replay).
- Undo steps run in reverse order of the completed steps.
- Non-undoable steps (`undo = none`) go last. The first one is the point of no return: after it the saga only moves forward. A non-undoable step must not fail on business grounds; its only failures are transient and are retried until they succeed. The runner refuses to register a type where a step with `undo` follows a step without one.

**Step contract**. Each step implements four methods. `run` is the `action` and `compensate` is the `undo` of the table above; the hooks add decisions.
```rust
#[async_trait]
trait Step {
    async fn run(&self, ctx: &mut Ctx) -> Result<Output, StepError>;
    async fn compensate(&self, ctx: &Ctx) -> Result<(), StepError>;                    // default: no-op (`undo = none`)
    fn on_success(&self, ctx: &mut Ctx, out: Output) -> Next { ctx.store(out); Next::Step(self.next_in_order()) }
    fn on_failure(&self, ctx: &Ctx, err: &StepError, attempt: u32) -> Failure { Failure::Compensate }
}
enum Next    { Step(StepId), Done, Wait }
enum Failure { Retry, Compensate, Skip, Fail }
#[derive(thiserror::Error, Debug)]
enum StepError { #[error("retryable: {0}")] Retryable(String), #[error("permanent: {0}")] Permanent(String) }
```
- `on_success` stores the step output in the saga context (`sagas.payload`) and returns `Next`: `Step(id)` (any step, so a saga can branch on earlier outputs), `Done` (final status `DONE`), or `Wait` (park until a reply arrives).
- `on_failure` handles the failed step and returns `Failure`: `Retry` (same key, bounded by `max_attempts`; past the limit the runner treats it as `Compensate`), `Compensate` (set `failure_reason`, start the undo), `Skip` (continue as if done, nothing stored), `Fail` (stop, status `FAILED`). `StepError::Retryable` is transient and `Permanent` is a business rejection (see Failure handling).
- `compensate` is not `on_failure`: `on_failure` decides what happens after the failed step, `compensate` undoes steps that already succeeded, in reverse order, and must be idempotent (key `{saga_id}-{step}-undo`).
- Hooks run inside the runner transaction (step 6) and write the `saga_steps` row and the `sagas` update. They are synchronous and do no external side effects; to call something, add a step.
- Defaults: `on_success` -> next step in order (`Done` after the last); `on_failure` -> `Compensate`.
- A `Wait` step leaves the saga `IN_PROGRESS`. The reply handler (Kafka consumer for the reply event) loads the row `FOR UPDATE` and calls `on_success` or `on_failure` for that step; the sweeper does not resume it before `SAGA_STALE_AFTER`.

**State table** (standard DDL, copied into each service's init migration like the outbox)
```sql
CREATE TABLE sagas (
  saga_id        uuid        PRIMARY KEY,
  type           text        NOT NULL,                 -- which step list
  current_step   text        NOT NULL,                 -- last completed step, or 'START'
  status         text        NOT NULL DEFAULT 'IN_PROGRESS',
  failure_reason text,                                 -- set = run undo steps in reverse
  payload        jsonb       NOT NULL,                 -- input and step outputs
  updated_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT sagas_status_chk CHECK (status IN ('IN_PROGRESS','DONE','FAILED','REVERSED'))
);
CREATE INDEX sagas_active_idx ON sagas (updated_at) WHERE status = 'IN_PROGRESS';  -- sweeper
```
**Step history** (`saga_steps`, append-only, part of the standard DDL)
```sql
CREATE TABLE saga_steps (
  saga_id uuid        NOT NULL,
  step    text        NOT NULL,                        -- action or undo step name
  status  text        NOT NULL,
  ts      timestamptz NOT NULL DEFAULT now(),
  payload jsonb,
  CONSTRAINT saga_steps_status_chk CHECK (status IN ('started','done','failed','compensated'))
);
CREATE INDEX saga_steps_saga_idx ON saga_steps (saga_id, ts);
```
- Rows are only inserted, never updated or deleted (the runtime role has no UPDATE/DELETE). The runner appends `started`, then `done` or `failed` for a forward step, and `compensated` for an undo, in the same transaction as the `sagas` update (step 6 below).
- History of one saga: `SELECT * FROM saga_steps WHERE saga_id = $1 ORDER BY ts;`
- `sagas` stays the current-state row; `saga_steps` is the audit trail.

- `status` final values: `DONE` (all steps ran), `FAILED` (failed with nothing to undo), `REVERSED` (failed and the done steps were undone). Final rows are immutable.
- `failure_reason` set with `IN_PROGRESS` means: run the undo steps, not the forward steps. There is no separate compensating status.
- Services keep business fields in their own tables and link by `saga_id` (for transfers, `saga_id = transfer_id`).

**Runner loop** (one saga per Kafka command message; command key and payload carry `saga_id`)
1. Consume the command (group per saga type). Do not commit the offset yet.
2. Begin a transaction: `SELECT ... FROM sagas WHERE saga_id = $1 FOR UPDATE`. Missing row -> DLQ.
3. Final status -> commit the offset, done.
4. Pick the step: forward (next step after `current_step`) or, if `failure_reason` is set, the next undo (the latest completed step that has an undo).
5. Run the step with its key. Calls are idempotent, so running inside the open transaction is safe; the lock is held only for the call's timeout (every call has one).
6. Update the row in the same transaction: `current_step`, `status`, `failure_reason`, `payload`, `updated_at`. A per-type hook runs in this transaction to update business tables and `outbox::enqueue` events. Commit.
7. Repeat from 2 until the status is final, then **commit the Kafka offset last** (ack after the step is committed; a pod that dies before this gets the command redelivered).

**Failure handling**
- Transient error (timeout, `5xx`, retriable codes): retry the same step with the same key, exponential backoff with jitter, up to `max_attempts`. The row lock is released between attempts.
- After attempts are exhausted for a forward step, it counts as permanent.
- Permanent error (business rejection): save `failure_reason` and run the done steps' undo in reverse. No completed step with an undo -> status `FAILED`. After the last undo -> `REVERSED`.
- Undo failures are retried without limit (the saga holds its partition) and raise an alert (`saga_undo_stuck`).
- A permanent failure in a non-undoable step is a bug: the runner alerts and stops the saga for manual action.

**Crash recovery**. State is saved only in step 6, the offset only in step 7.

| Crash point | State on restart | Result |
|---|---|---|
| 1. Before the step ran | row unchanged, offset not committed | command redelivered, the step runs |
| 2. After the step, before the row is saved | callee did the work, row unchanged | redelivered, the step runs again with the same key, the callee returns the stored response (replay) |
| 3. After the row is saved, before the offset commit | row is ahead of the offset | redelivered, the runner loads the row and continues from `current_step` (or commits the offset if final) |

- Sweeper: a task per service runs `SELECT saga_id FROM sagas WHERE status = 'IN_PROGRESS' AND updated_at < now() - $stale` (default 60 s) and re-enqueues the command through the outbox. It covers a lost or dead-lettered command, or a consumer group that moved past the offset. Re-enqueueing is safe: the runner is idempotent.
- Row lock: two pods working on one saga (redelivery during a slow step, sweeper racing a live worker) serialize on `FOR UPDATE`. The second one reloads the row, sees the saved progress, and continues or exits. Steps never run concurrently for one saga.

**Config**: `SAGA_MAX_ATTEMPTS` (default 5, about 30 s total), `SAGA_STALE_AFTER` (default 60 s), `SAGA_SWEEP_INTERVAL` (default 15 s), `SAGA_UNDO_MAX_BACKOFF` (default 30 s).

**Observability**: every log line and trace span written while handling a saga carries `saga_id` (plus `type` and `step`), via a tracing span opened by the runner per command (`common/003-telemetry`).

**Metrics**: step latency and attempts by type/step, sagas by final status, in-progress count, oldest in-progress age, sweeper re-enqueues, `saga_undo_stuck` gauge.

## Dependencies
- `common/001-money-and-errors`, `common/003-telemetry`, `common/004-events-kafka`, `common/005-outbox`, `platform/002-workspace-and-compose`.

## Acceptance criteria
- [ ] A saga with N steps runs them in order; `current_step` and `updated_at` advance in the same transaction as the step result.
- [ ] Every call carries key `{saga_id}-{step}` (undo: `{saga_id}-{step}-undo`).
- [ ] Transient errors retry with backoff and the same key; exhausted retries on a forward step start the undo.
- [ ] A permanent error undoes completed steps in reverse and ends `REVERSED`, or `FAILED` when nothing was done.
- [ ] Registering a step list with an undoable step after a non-undoable one is rejected.
- [ ] Crash at each of the 3 crash points (fault hook) ends in the same final state with each effect applied once.
- [ ] Two workers on one saga never run a step concurrently (row lock).
- [ ] The sweeper re-enqueues an `IN_PROGRESS` saga with stale `updated_at` and ignores final or fresh ones.
- [ ] The offset is committed only after the saved row is final.
- [ ] `saga_steps` is append-only and the history query returns every step of a saga in order, including `compensated` rows after an undo.
- [ ] The same `saga_id` appears in every log line and span for a saga.
- [ ] `on_success` output is stored in the saga context, and a returned `Next::Step(id)` jumps to that step (branching), `Done` ends `DONE`.
- [ ] Default hooks: `on_success` goes to the next step in order, `on_failure` starts compensation.
- [ ] `on_failure` returning `Retry` retries up to `max_attempts` then compensates; `Skip` continues; `Fail` ends `FAILED`.
- [ ] `StepError` separates retryable from permanent: only `Retryable` reaches `Retry`.
- [ ] `compensate` runs in reverse order only for succeeded steps, and replaying it with the same key has no second effect.
- [ ] Hooks run in the runner transaction: the `saga_steps` row and `sagas` update commit together, and a hook failure rolls both back.
- [ ] A `Wait` step leaves the saga `IN_PROGRESS`; the reply handler resumes it through `on_success` or `on_failure`.

## Test plan
- Unit: step ordering, undo selection, backoff schedule, step-list validation.
- Integration (testcontainers Postgres + Kafka, stub callees): happy path, transient then success, permanent with undo, fault injection at each crash point, concurrent workers on one saga, sweeper, kill-mid-step (worker killed during a step, restarted: one effect, saga final), `saga_steps` history and append-only grant.
