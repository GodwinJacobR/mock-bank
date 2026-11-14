# Saga runner

Status: draft

## Goal
Library crate `common/saga` (`bank-saga`): an orchestrator for multi-step workflows across services. A service defines a saga type as a step list; the orchestrator drives it by sending commands to other services over Kafka and reacting to their replies, with idempotent steps, persisted state, retries, undo in reverse, and crash recovery. First user: transfers (`transfers/001-service`).

## Scope (in)
- Step definition, generic `sagas` state table DDL, the start transaction, the reply handler, failure handling, crash recovery (sweeper, row lock).
- Command/reply messages over Kafka (`common/004-events-kafka`) and the outbox (`common/005-outbox`).
- Metrics hooks.
- Step history table (`saga_steps`) and `saga_id` on logs and trace spans.

## Out of scope
- Business logic of any step (owned by the service that receives the command).
- Cross-service distributed transactions (2PC): the state table lives in the orchestrating service's schema.
- Multi-saga parent/child chains.
- Synchronous HTTP calls from the orchestrator to the participant services (steps are async commands).

## Design

**Orchestration model**. The orchestrator (the service owning the saga) never calls a participant directly. A step is a command and a reply.
- The orchestrator writes a command row to its outbox; the relay publishes it to the participant's command topic (for accounts: `accounts.commands`, key = `saga_id`).
- The participant consumes the command, **dedupes by `idempotency_key`** (same key: it re-sends the stored reply, no second effect), does the work, publishes the reply to the reply topic (`accounts.replies`, key = `saga_id`), and **acks (commits the offset of) the command only after the reply is published**. A crash before that redelivers the command, and the dedupe makes it safe.
- The orchestrator consumes the replies and advances the saga.

Command (topic `accounts.commands`):
```json
{ "saga_id": "6f1c2a40-0000-4000-8000-000000000001", "step": "debit", "cmd": "debit",
  "account": "00000000-0000-4000-8000-0000000000a1", "amount": 100,
  "idempotency_key": "59bd9d3d-f1a7-5162-902f-0138792ce901" }
```
Reply (topic `accounts.replies`):
```json
{ "saga_id": "6f1c2a40-0000-4000-8000-000000000001", "step": "debit", "result": "ok", "code": null }
```
`result` is `ok` or `failed`; `code` is the business reason on `failed` (e.g. `insufficient_funds`), else null. Replies carry no other data the saga needs beyond `result` and `code`.

**Step list**. A saga type is an ordered list of steps.

| Field | Meaning |
|---|---|
| `name` | step name, unique within the type and without `:` (the `{name}:undo` form is reserved); stored in `sagas.current_step` |
| `command` | builds the command (`cmd`, `account`, `amount`) from the saga `context` |
| `undo` | builds the undo command, or `none` |
| idempotency key | deterministic: `UUIDv5(namespace = saga_id, name = step)` for the action; `UUIDv5(namespace = saga_id, name = "{step}:undo")` for the undo. Carried in the command |
| retry policy | `max_attempts`, `base_backoff`, `max_backoff`, jitter; undo steps default to unlimited attempts (cap 30 s) with an alert |

Rules
- Every command is idempotent: same key, same effect (the participant returns the stored reply on replay). The key is a pure function of `(saga_id, step)`, so every re-send (retry, sweeper, redelivery) carries the same key.
- Undo steps run in reverse order of the completed steps.
- Non-undoable steps (`undo = none`) go last. The first one may still be rejected (the earlier undoable steps are then undone); once it has succeeded it is the point of no return and the saga only moves forward. Any non-undoable step after the first must not fail on business grounds; its only failures are transient (timeout) and are retried until they succeed. A timeout on the first non-undoable step is also retried without limit (the outcome is unknown and it may have applied). The orchestrator refuses to register a type where a step with `undo` follows a step without one.

**Step contract**. Each step implements these methods. They are synchronous and pure: they decide, they do not call anything.
```rust
trait Step {
    fn command(&self, ctx: &Ctx) -> Command;                                    // built into an outbox row
    fn undo_command(&self, ctx: &Ctx) -> Option<Command> { None }               // default: no-op (`undo = none`)
    fn on_success(&self, ctx: &mut Ctx, reply: &Reply) -> Next { Next::Step(self.next_in_order()) }
    fn on_failure(&self, ctx: &Ctx, reply: &Reply, attempt: u32) -> Failure { Failure::Compensate }
}
enum Next    { Step(StepName), Done, Wait }
enum Failure { Retry, Compensate, Skip, Fail }
```
- `on_success` stores what it needs from the reply in the saga `context` and returns `Next`: `Step(name)` (any step of the type, looked up by name in the saga type's step list, so a saga can branch on earlier outputs), `Done` (status `COMPLETED`), or `Wait` (park without sending a command, see Waiting). After the last step the default is `Done`.
- `on_failure` handles a `failed` reply (or a timeout, code `timeout`) for the current step and returns `Failure`: `Retry` (same command, same key, bounded by `max_attempts`; past the limit the orchestrator treats it as `Compensate`), `Compensate` (set `failure_reason`, start the undo), `Skip` (continue as if done, nothing stored), `Fail` (stop, status `FAILED`).
- `undo_command` undoes steps that already succeeded, in reverse order, with key `UUIDv5(saga_id, "{step}:undo")`.
- Hooks run inside the reply-handler transaction and write the `saga_steps` row, the `sagas` update and the next outbox row. They do no external side effects; to call something, add a step.
- Defaults: `on_success` -> next step in order; `on_failure` -> `Compensate`.

**State table** (standard DDL, copied into each service's init migration like the outbox)
```sql
CREATE TABLE sagas (
  saga_id        uuid        PRIMARY KEY,
  type           text        NOT NULL,                 -- which step list
  status         text        NOT NULL DEFAULT 'RUNNING',
  current_step   text        NOT NULL,                 -- step name; RUNNING: current step (`debit`); COMPENSATING: step being undone (`debit:undo`)
  context        jsonb       NOT NULL,                 -- input and values kept from replies
  failure_reason text,                                 -- set when compensation starts
  deadline       timestamptz,                          -- when the awaited reply is overdue; NULL = not waiting
  updated_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT sagas_status_chk CHECK (status IN ('RUNNING','COMPENSATING','COMPLETED','REVERSED','FAILED'))
);
CREATE INDEX sagas_active_idx ON sagas (updated_at) WHERE status IN ('RUNNING','COMPENSATING');  -- sweeper
CREATE INDEX sagas_deadline_idx ON sagas (deadline) WHERE deadline IS NOT NULL;                 -- sweeper
```
**Step history** (`saga_steps`, append-only, part of the standard DDL)
```sql
CREATE TABLE saga_steps (
  saga_id uuid        NOT NULL,
  step    text        NOT NULL,                        -- action name, or "{name}:undo" for an undo
  status  text        NOT NULL,
  ts      timestamptz NOT NULL DEFAULT now(),
  payload jsonb,                                       -- command sent, reply received, attempt number
  CONSTRAINT saga_steps_status_chk CHECK (status IN ('started','done','failed','compensated'))
);
CREATE INDEX saga_steps_saga_idx ON saga_steps (saga_id, ts);
```
- Rows are only inserted, never updated or deleted (the runtime role has no UPDATE/DELETE). `started` is written when a command is queued (payload: command, `attempt`), then `done` or `failed` (payload: reply) for a forward step; an undo writes `started` and `compensated` under the name `{step}:undo`. Each row is written in the same transaction as the `sagas` update and the outbox row.
- History of one saga: `SELECT * FROM saga_steps WHERE saga_id = $1 ORDER BY ts;`
- `sagas` is the current-state row; `saga_steps` is the audit trail.

Statuses
- `RUNNING`: forward steps. `COMPENSATING`: undo commands in reverse. `COMPLETED`: all steps done. `REVERSED`: failed and the done steps were undone. `FAILED`: the first step failed (nothing to undo) or `on_failure` returned `Fail`. Final rows (`COMPLETED`, `REVERSED`, `FAILED`) are immutable.
- Services keep business fields in their own tables and link by `saga_id` (for transfers, `saga_id = transfer_id`).

**Start transaction** (called by the service endpoint, e.g. `POST /transfers`). In one transaction:
1. insert `sagas` (`RUNNING`, `current_step` = first step name, `context` = input, `deadline = now() + step timeout`);
2. insert `saga_steps` (first step, `started`, attempt 1);
3. insert the outbox command row (topic of the participant, key = `saga_id`, payload = command);
4. the service's own rows (business table, domain event outbox rows such as `transfer.created`).
Commit and return `202` with the `saga_id`. Nothing is sent from the request path; the relay publishes the command (`published_at IS NULL` rows are the unsent ones).

**Reply handler** (Kafka consumer of the reply topic, group per saga type; replies are keyed by `saga_id`)
1. Consume the reply. Do not commit the offset yet.
2. Begin a transaction: `SELECT ... FROM sagas WHERE saga_id = $1 FOR UPDATE`. Missing row -> DLQ (`<reply topic>.dlq`, e.g. `accounts.replies.dlq`).
3. Drop the reply (commit the offset, nothing written) if the saga is final, if `reply.step` is not the current step (i.e. `reply.step == sagas.current_step`: the step name for `RUNNING`, its `{name}:undo` for `COMPENSATING`), or if that step already has a `done`/`compensated` row.
4. Otherwise, in this one transaction: call `on_success` or `on_failure` (by `result`); append the `saga_steps` row; update `sagas` (`current_step`, `status`, `context`, `failure_reason`, `deadline`, `updated_at`); insert the next outbox command row (next step, retry, or next undo) with its `started` row; run the service's hook (business table mirror, domain event outbox rows). Last step done -> `COMPLETED`. Commit.
5. Commit the Kafka offset last. A pod that dies before this gets the reply redelivered; step 3 then drops it as already applied.

**Failure handling**
- `failed` reply: `on_failure(reply, attempt)`. `Compensate` sets `failure_reason`, status `COMPENSATING`, and queues the undo command of the latest completed step that has one (reverse order). If the failure was a timeout on an undoable step, that step is undone first too (its outcome is unknown). Each undo reply marks that step `compensated` and queues the next; after the last undo the status is `REVERSED`. If nothing needs undoing (the first step failed with a `failed` reply), the status is `FAILED`.
- `Retry`: the same command with the same key is re-sent after an exponential backoff with jitter, up to `max_attempts`: the handler records a `failed` row, sets `deadline = now() + backoff`, and the sweeper re-publishes it when due (see Sweeper). Past `max_attempts` it counts as `Compensate`.
- Undo `failed` replies are retried until they succeed (backoff capped at 30 s) and raise an alert (`saga_undo_stuck`). Compensation is never abandoned.
- A business failure in a non-undoable step after the point of no return is a bug: the orchestrator alerts and stops the saga for manual action.
- Timeout (no reply by `deadline`) is handled as a failure with code `timeout`; see Sweeper.

**Waiting and deadline**. Every queued command sets `deadline = now() + SAGA_STEP_TIMEOUT` (default 30 s). `Next::Wait` parks the saga with a `deadline` and no command (e.g. waiting for an external event); the reply or event handler calls `on_success`/`on_failure` for the parked step. `deadline` is cleared (NULL) when a step completes.

**Sweeper**. A task per service runs every `SAGA_SWEEP_INTERVAL`:
- Stale: `status IN ('RUNNING','COMPENSATING') AND deadline IS NULL AND updated_at < now() - $stale` (default 60 s): re-publish the command of the current step (a new outbox row, same key). Covers a lost command or reply, or a consumer group that moved past the offset.
- Past deadline: `deadline < now()`. If the last `saga_steps` row of the current step is `failed` (backoff over), re-publish the command. If it is `started` (no reply), call `on_failure` with code `timeout`: `Retry` re-publishes the same command with the same key (the participant dedupes and returns the stored outcome), bounded by `max_attempts`; `Compensate` starts the undo. Every re-publish writes a new `started` row (attempt + 1) and sets `deadline = now() + SAGA_STEP_TIMEOUT`. For undo steps and non-undoable steps a timeout always retries.
- The sweeper re-publishes the command of `sagas.current_step` (for `COMPENSATING`, the undo command of the step named `{name}` in `{name}:undo`).
- Re-publishing is safe: commands are idempotent and the reply handler drops duplicates.
- **Late replies**: a reply that arrives after a timeout, a retry or compensation is checked against the saga state by the reply handler (step 3). It is applied only if it matches the current step and that step is not already done; otherwise dropped. Because a timeout re-sends the same key rather than guessing, the outcome of a timed-out step is learned from the participant's stored reply, not assumed. The participant must reject a forward command whose `{step}:undo` key it has already processed (reply `failed`, code `compensated`), so a late action cannot land after its undo. An undo for an action it never applied is recorded and answered `ok` with no effect.

**Crash recovery**. State changes only in the start transaction and the reply-handler transaction; the reply offset is committed last.

| Crash point | State on restart | Result |
|---|---|---|
| Orchestrator dies before the start tx commits | nothing stored | client retries with its idempotency key |
| Orchestrator dies after the start tx, before the relay publishes | outbox row has `published_at IS NULL` | the relay publishes it on restart |
| Relay dies after Kafka accepts the command, before `published_at` is set | row still unsent | command published twice; the participant dedupes by key |
| Participant dies before replying | command not acked | redelivered; processed once by key |
| Participant dies after replying, before acking | reply already sent | redelivered; dedupe re-sends the stored reply; the reply handler drops the duplicate |
| Orchestrator dies mid reply handler | tx rolled back, offset not committed | reply redelivered, handled again |
| Orchestrator dies after the tx, before the offset commit | saga is ahead of the offset | reply redelivered; step 3 drops it |
| Reply lost, pod gone for good | saga stuck on a step | sweeper re-publishes (stale or past deadline) |

- Row lock: two pods working on one saga (reply redelivery racing the sweeper) serialize on `FOR UPDATE`. The second one reloads the row, sees the saved progress, and drops or continues. A saga moves one step at a time.

**Config**: `SAGA_MAX_ATTEMPTS` (default 5), `SAGA_STEP_TIMEOUT` (default 30 s), `SAGA_STALE_AFTER` (default 60 s), `SAGA_SWEEP_INTERVAL` (default 15 s), `SAGA_UNDO_MAX_BACKOFF` (default 30 s).

**Observability**: every log line and trace span written while handling a saga carries `saga_id` (plus `type` and `step`), via a tracing span opened per reply, per start and per sweep (`common/003-telemetry`). The `saga_id` is also in the command and reply messages, so a participant can log it.

**Metrics**: step latency (command queued to reply) and attempts by type/step, sagas by final status, running and compensating counts, oldest running age, timeouts, sweeper re-publishes, outbox unsent count, `saga_undo_stuck` gauge.

## Worked example: debit A, credit B, 100

Fixed ids: accounts `A = 00000000-0000-4000-8000-0000000000a1`, `B = 00000000-0000-4000-8000-0000000000b1`. Steps: `debit` (undo: credit back), `credit` (no undo). Keys are `UUIDv5(namespace = saga_id, name = step)`.

**1. Happy path**, `saga_id = 6f1c2a40-0000-4000-8000-000000000001`

| # | Event | Message / state |
|---|---|---|
| 1 | Start tx commits (`202`) | `sagas`: `RUNNING`, `current_step = debit`; `saga_steps`: `debit started`; outbox: command below |
| 2 | Relay publishes to `accounts.commands` (key = saga_id) | `{"saga_id":"6f1c2a40-0000-4000-8000-000000000001","step":"debit","cmd":"debit","account":"…a1","amount":100,"idempotency_key":"59bd9d3d-f1a7-5162-902f-0138792ce901"}` |
| 3 | Accounts debits A, replies, then acks | `accounts.replies`: `{"saga_id":"6f1c2a40-0000-4000-8000-000000000001","step":"debit","result":"ok","code":null}` |
| 4 | Reply handler (one tx) | `saga_steps`: `debit done`, `credit started`; `current_step = credit`; outbox: `{"saga_id":"…0001","step":"credit","cmd":"credit","account":"…b1","amount":100,"idempotency_key":"0cecbc72-c4bc-5ba5-9fc3-5568ebc9bf4b"}` |
| 5 | Accounts credits B | reply `{"saga_id":"…0001","step":"credit","result":"ok","code":null}` |
| 6 | Reply handler | `saga_steps`: `credit done`; `sagas.status = COMPLETED` |

**2. Credit rejected, compensation**, `saga_id = 6f1c2a40-0000-4000-8000-000000000002` (keys: debit `2c0acf23-5500-55d3-ad94-eca3a7a29eb9`, credit `00417a0e-6a0b-5426-97a6-479fe0f19c03`, debit undo `15c540dd-baab-5df6-bef8-f7f3757acde7`)
1. Debit as above succeeds; `credit` command is queued.
2. Accounts replies `{"saga_id":"6f1c2a40-0000-4000-8000-000000000002","step":"credit","result":"failed","code":"account_closed"}`.
3. Reply handler (one tx): `on_failure` -> `Compensate`; `saga_steps`: `credit failed`, `debit:undo started`; `sagas`: `status = COMPENSATING`, `current_step = debit:undo`, `failure_reason = account_closed`; outbox: `{"saga_id":"6f1c2a40-0000-4000-8000-000000000002","step":"debit:undo","cmd":"undo_debit","account":"…a1","amount":100,"idempotency_key":"15c540dd-baab-5df6-bef8-f7f3757acde7"}`.
4. Accounts restores A, replies `{"saga_id":"…0002","step":"debit:undo","result":"ok","code":null}`.
5. Reply handler: `saga_steps`: `debit:undo compensated`; `status = REVERSED`. (If the `debit` itself had failed, the status would be `FAILED` with no undo.)

**3. Pod dies, recovery**, `saga_id = 6f1c2a40-0000-4000-8000-000000000003` (keys: debit `48d586b7-421f-533b-ab30-e760e67085ca`, credit `af978dd4-8242-58b4-ba1c-67e9896d5e19`)
1. Accounts debits A and publishes the debit reply; the orchestrator pod is killed before committing the reply offset (or before its transaction commits).
2. A restarted pod gets the reply redelivered. If the first transaction committed, `debit` is already `done` and step 3 of the handler drops it. If it did not commit, the reply is handled now and the `credit` command is queued.
3. If the reply was lost instead, the saga stays `RUNNING` on `debit` with `deadline` set. When the deadline passes the sweeper calls `on_failure` with `timeout` -> `Retry`: it re-publishes the same debit command (same key `48d586b7-…`). Accounts finds the key, re-sends the stored reply, and does not debit twice. The saga continues to `credit` and `COMPLETED`.

## Dependencies
- `common/001-money-and-errors`, `common/003-telemetry`, `common/004-events-kafka`, `common/005-outbox`, `platform/002-workspace-and-compose`.

## Acceptance criteria
- [ ] The start transaction writes `sagas`, the first `saga_steps` row, the outbox command row and the service's own rows together (a rollback leaves none) and returns `202` with the `saga_id`; no participant is called from the request.
- [ ] A saga with N steps runs them in order; `current_step`, `updated_at` and the next outbox row advance in the same transaction as the reply result.
- [ ] Every command carries `{saga_id, step, cmd, account, amount, idempotency_key}`, key `UUIDv5(saga_id, step)` (undo: `UUIDv5(saga_id, "{step}:undo")`), identical on every re-send.
- [ ] The command is published to the participant topic with key = `saga_id`; the participant dedupes by key, replies `{saga_id, step, result, code}`, and acks the command only after the reply.
- [ ] The reply handler loads the saga `FOR UPDATE` and drops a reply whose step is not the current step or is already done, writing nothing.
- [ ] A `failed` reply with the default `on_failure` sets `COMPENSATING`, queues undo commands in reverse, marks each `compensated`, and ends `REVERSED`; a first-step failure ends `FAILED`.
- [ ] Undo failures are retried until success and raise `saga_undo_stuck`.
- [ ] `Retry` re-sends the same command and key with backoff up to `max_attempts`, then compensates; `Skip` continues; `Fail` ends `FAILED`.
- [ ] The last step done ends `COMPLETED`; a returned `Next::Step(name)` jumps to the step with that name (branching).
- [ ] Registering a step list with an undoable step after a non-undoable one is rejected.
- [ ] A reply for a saga past a timeout or in compensation is checked against state: a stale reply never changes the saga.
- [ ] A saga past its `deadline` is retried (same key) or compensated by the sweeper per `on_failure`; a saga with `deadline IS NULL` and stale `updated_at` has its current command re-published; final or fresh sagas are ignored.
- [ ] Each row of the crash table ends in the same final state with each effect applied once.
- [ ] Two reply handlers on one saga never apply a step concurrently (row lock).
- [ ] The reply offset is committed only after the transaction.
- [ ] `saga_steps` is append-only and the history query returns every step of a saga in order, including `{step}:undo` `compensated` rows.
- [ ] The same `saga_id` appears in every log line and span for a saga.
- [ ] Hooks run in the handler transaction: the `saga_steps` row, `sagas` update and outbox row commit together, and a hook failure rolls all back.
- [ ] A `Wait` step sets `deadline` and sends no command; the reply/event handler resumes it through `on_success` or `on_failure`.

## Test plan
- Unit: step ordering, undo selection, key derivation (UUIDv5 vectors for the worked example), backoff schedule, step-list validation (names unique, no `:`), `current_step` lookup by name incl. `:undo`, reply-drop rules (`reply.step != current_step`).
- Integration (testcontainers Postgres + Kafka, a stub participant that dedupes by key): worked example 1 (happy path), 2 (credit rejected, `REVERSED`; first step rejected, `FAILED`), 3 (kill the orchestrator mid handler, redelivery, sweeper retry on timeout), duplicate and late replies, duplicate command (relay crash), concurrent handlers on one saga, sweeper stale and deadline paths, undo failing then succeeding, `saga_steps` history and append-only grant, start-tx rollback, `Retry`/`Skip`/`Fail`/`Wait` and branching, hook failure rollback, `saga_id` in logs and spans.
