# Telemetry

Status: draft

## Goal
Library crate `common/telemetry` (`bank-telemetry`): one-call logging setup, a metrics endpoint helper and health endpoints helper, used by every service.

## Scope (in)
- `init_tracing()`: JSON logs to stdout.
- `/metrics` helper (Prometheus exporter).
- `/healthz` and `/readyz` router helper.

## Out of scope
- OTLP export, collector, Tempo, Grafana (phase 5; the crate leaves a hook for `tracing-opentelemetry`).
- Business metrics definitions (owned by each crate/service, e.g. outbox lag in `common/outbox`).

## Design
- `init_tracing(service_name)`: `tracing-subscriber` with JSON formatter, `RUST_LOG` env filter (default `info`), `service` field on every line. Idempotent (second call is a no-op).
- `metrics_router(handle)`: `GET /metrics` serving the `metrics-exporter-prometheus` text format; `install_metrics()` returns the handle. Includes an HTTP request counter/latency layer (`tower` layer) labelled by route and status.
- `health_router(checks)`: `GET /healthz` returns `200 {"status":"ok"}` (no dependency checks); `GET /readyz` runs registered `ReadyCheck` async closures (DB ping, Kafka metadata) and returns `200` or `503 {"status":"not_ready","failed":["db"]}`.
- Trace context: `traceparent` extraction/injection helpers for HTTP (axum layer) and a plain `HeaderMap` form that `common/events` uses for Kafka headers.

## Dependencies
- `platform/workspace-and-compose`.

## Acceptance criteria
- [ ] Log lines are single-line JSON with `timestamp`, `level`, `service`, `target`, `message`.
- [ ] `/metrics` returns Prometheus text including the request counter.
- [ ] `/healthz` is 200 with no dependencies up; `/readyz` is 503 when a registered check fails.
- [ ] A `traceparent` injected then extracted yields the same trace id.

## Test plan
- Unit tests for readiness aggregation and traceparent round trip.
- Integration test: start a router with a failing check and assert status codes; scrape `/metrics`.
