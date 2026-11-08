# Scale and Portability

Status: draft

## Goal
- The design grows to about 1M req/s by adding instances and shards, not by redesign.
- The same image runs on Compose, Kubernetes and managed services (RDS, MSK, ElastiCache) with config-only changes.

## Scope (in)
This spec covers P0 (portability) and P1 (shard-ready code) only.
- Requirements R1-R6 below.
- Portability contract: env vars, images, health, jobs, Compose files.
- Design notes for later phases (cross-shard transfers, hot account, reads), recorded so P0/P1 code does not block them.

## Out of scope
- Actually running 1M req/s. Measure per-unit capacity and extrapolate; the local phase-5 target stays 200 req/s.
- Multi-region, real payment rails.
- Building sharding now.
- Follow-up specs: P2 k8s manifests/overlays, P3 k6 load tests, P4 actual sharding.

## Requirements
- R1 Stateless services: no local state, any instance serves any request.
- R2 Shard key: every ledger row carries `account_id`. 256 logical shards map to N physical primaries via `hash(account_id)`. A transfer within one shard is one DB transaction.
- R3 Idempotency: `Idempotency-Key` per customer on writes; `event_id` dedup on consumers.
- R4 Limits: timeouts on every network call, bounded pools and queues, overload returns 503/429 with `Retry-After`.
- R5 Config: all config from env vars, no hosts in code, secrets via env or mounted files.
- R6 Jobs: migrations, role bootstrap and topic creation run as separate one-shot jobs.

## Design

**Scale math** (assumptions to verify in P3)
- 1M req/s = 900k reads + 100k transfers/s.
- 90% Redis hit leaves 90k reads/s for about 9 replicas.
- About 3-5k transfers/s per primary gives about 25 primaries; start with 32 shards.
- Kafka about 150 MB/s, about 6 brokers at RF3.

**Key decisions**
- Cross-shard transfers: accepted with delayed credit. Debit the source into a per-shard clearing account and write an outbox event; the destination shard credits idempotently on the event. Each shard sums to zero. Designed now, implemented in P4. Rejected: 2PC and distributed SQL (Citus/Cockroach), for latency and no plain RDS.
- Treasury hot account: split into K sub-accounts, random pick per transfer; audit sums them.
- Reads: the transfer path reads the primary only. Balance and history read Redis, then a read replica. The transfer response returns the new balance.
- Kafka: partition key `account_id`, 64+ partitions.
- Backpressure: gateway concurrency limit and load shedding; small DB pools plus PgBouncer or RDS Proxy.
- Rate limiting: Redis token bucket per user/IP plus an in-process per-pod limit.
- Observability: RED metrics, `shard` label, outbox and consumer lag, pool wait. SLO: p99 < 200 ms, errors < 0.1%.
- Deploy tool: Kustomize (base plus kind/cloud overlays, in P2) with Compose for local. Helm rejected for now. The cloud overlay is AWS-only; code stays provider-neutral via env vars.

**Portability contract**
- Env vars: `PORT`, `DATABASE_URL` (runtime role, `sslmode` in the URL), `MIGRATE_DATABASE_URL` (owner role, jobs only), `REDIS_URL`, `KAFKA_BROKERS`, `KAFKA_SECURITY_PROTOCOL`, `KAFKA_SASL_*`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `JWT_SECRET`, `ADMIN_TOKEN`, `SHARD_MAP` (later), `RUST_LOG`.
- Images: one per service, tagged by git sha, non-root, multi-arch. Each binary has `serve` and `migrate` subcommands.
- Health: `/healthz` (liveness, no deps), `/readyz` (DB and Kafka). SIGTERM drains in-flight work within 30s.
- Jobs: `bootstrap` (roles and schemas, once, as admin); `<svc> migrate` (Compose `depends_on: service_completed_successfully`, k8s Job); topic creation (Kafka auto-create off).
- Logs: JSON to stdout; no volumes in service containers.
- Compose: `docker-compose.yml` (everything local) and `compose.external.yml` (no postgres/kafka/redis containers; URLs point to managed services).

## Dependencies
- `platform/002-workspace-and-compose`, `common/003-telemetry`, `common/004-events-kafka`, `common/005-outbox`, `ledger/002-schema`.

## Acceptance criteria
- [ ] Compose to kind to RDS changes only env vars and secrets; grep finds no hostnames in code.
- [ ] Killing a pod loses no in-flight requests (within timeout) and no events.
- [ ] Overload returns 503/429 with `Retry-After`.
- [ ] Every ledger row has `account_id` and the shard mapping function exists.
- Load-test report items are deferred to P3.

## Plan
1. P0: env-only config, `serve`/`migrate` subcommands, bootstrap and topic jobs, health/drain, image build, `compose.external.yml`.
2. P1: shard key on ledger rows and shard mapping, idempotency, timeouts/bounded pools/load shedding, rate limiting, metrics.

Follow-ups (separate specs): P2 Kustomize base and overlays, P3 k6 load tests, P4 sharding and cross-shard transfers.
