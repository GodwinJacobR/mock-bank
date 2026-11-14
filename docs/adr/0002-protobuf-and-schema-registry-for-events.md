# Protobuf and a schema registry for Kafka events

Kafka domain events will use Protobuf (Rust via prost) with a schema registry, in backward-compatible mode by default. This is introduced in a future milestone; M1 stays on [specs/common/004-events-kafka.md](../../specs/common/004-events-kafka.md) as is. The registry product (Confluent vs Apicurio) is left open for the spec.

## Considered Options

- Avro: rejected in favour of Protobuf.
- JSON Schema: rejected in favour of Protobuf.
