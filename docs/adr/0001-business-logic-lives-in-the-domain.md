# Business logic lives in the domain

All business logic lives in a pure domain module per context: aggregates, value objects, domain events, and domain errors, with no I/O, Kafka, SQL, or HTTP. Handlers, repositories, the outbox, and the gateway only orchestrate and contain no business-rule conditionals. DB constraints are allowed only as a safety net duplicating a rule already enforced in the domain.

## Considered Options

- Rules in the service/handler layer: rejected, rules get scattered and are hard to test without I/O.
- Rules in DB constraints/triggers: rejected, rules become invisible to domain code and hard to evolve.
