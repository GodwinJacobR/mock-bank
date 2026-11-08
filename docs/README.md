# Docs

Diagrams use Mermaid. Specs live in [../specs](../specs/README.md); repo knowledge in [../knowledge](../knowledge).

One folder per service, plus shared areas. Each service folder has a `README.md` (role, API, data owned, events, interaction diagram).

- [gateway/](gateway/README.md): public REST API, auth, routing.
- [accounts/](accounts/README.md): customers, accounts, payees, sessions.
- [ledger/](ledger/README.md): double-entry postings and balances.
- [transfers/](transfers/README.md): async transfer workflow.
  - [backpressure.md](transfers/backpressure.md): what happens when transfers are busy (rate limits, admission control, lag).
  - [ordering.md](transfers/ordering.md): running instructions in order (saga, compensation, Kafka key per account).
- [web/](web/README.md): server-rendered UI and admin console.
- [saga/](saga/README.md): generic saga runner: loop, status, crash recovery (3 crash points).
- [platform/system-flow.md](platform/system-flow.md): components, async transfer sequence, saga state.

Transfers are a separate service ([specs/transfers](../specs/transfers/001-service.md)) and the first saga type on the generic runner ([specs/common/006-saga](../specs/common/006-saga.md)); the ledger only does postings. The client gets `202` and reads the result by polling or SSE.

Each page marks what is in the specs ([Spec]) and what is still a proposal ([Proposal]). Updated by `/wrapup`.
