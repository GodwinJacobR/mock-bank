# Docs

Diagrams use Mermaid. Specs live in [../specs](../specs/README.md); repo knowledge in [../knowledge](../knowledge).

- [system-flow.md](system-flow.md): components, async transfer sequence (202, Kafka, saga), saga state diagram.
- [transfers-backpressure.md](transfers-backpressure.md): what happens when transfers are busy (rate limits, admission control, lag).
- [transfers-ordering.md](transfers-ordering.md): running instructions in order (saga, compensation, Kafka key per account).

Transfers are a separate service ([specs/transfers](../specs/transfers/001-service.md)); the ledger only does postings. The client gets `202` and reads the result by polling or SSE.

Each page marks what is in the specs ([Spec]) and what is still a proposal ([Proposal]). Updated by `/wrapup`.
