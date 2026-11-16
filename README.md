# Distributed Payments & Ledger Engine

A multi-service double-entry payments and ledger platform written in Go. Six services communicate over HTTP, gRPC (Protobuf), and NATS JetStream, backed by isolated per-service PostgreSQL databases, Redis, and OpenTelemetry observability.

## Architecture

Each service owns its own PostgreSQL database and communicates synchronously over gRPC (with mutual TLS between internal services) or asynchronously via a transactional outbox relayed to NATS JetStream durable consumers.

| Service | Ports | Responsibility |
| --- | --- | --- |
| `gateway` | `8080` (internal `9080`) | Public edge: JWT verification, Redis rate limiting, CORS, timeouts, and reverse proxy |
| `wallet` | `8081`, gRPC `9081` | Double-entry ledger, row-locked ACID transfers, holds, idempotency, FX conversion, reconciliation, and CSV statements |
| `auth` | `8082` | User accounts, Argon2id password hashing, EdDSA access tokens, and rotating refresh tokens |
| `transfer` | `8083` | Saga-orchestrated transfers (hold, capture/release, compensation), Redis velocity checks, risk rules, and manual review |
| `notification` | `8084` | Idempotent event consumers, exponential-backoff retries, and dead-letter queue handling |
| `webhook` | `8085` | Signed outbound webhook deliveries, delivery logs, and replay |

### Core Invariants

- **Integer Minor Units**: All monetary amounts are represented as `int64` minor units (cents), never floating-point numbers.
- **Zero-Sum Double-Entry Ledger**: Account balances are derived from append-only ledger entries, and database constraints enforce that every transaction's entries sum to zero per currency.
- **Concurrency & Deadlock Prevention**: Multi-account transfers acquire PostgreSQL row-level locks (`FOR UPDATE`) in deterministic account-ID order with bounded serialization retries.
- **Idempotency & Outbox**: Every money-moving write requires an idempotency key (backed by a Redis fast path and PostgreSQL persistence), and domain events are published via a transactional outbox relay to NATS JetStream.
- **Security Hardening**: Internal service-to-service gRPC uses mTLS with a local dev CA (`cmd/devca`), least-privilege PostgreSQL roles, and an audit log for manual risk reviews.

## Repository Map

```text
api/           .http request collections for manual testing
chaos/         game-day chaos scripts and saga fault injection
cmd/           service entrypoints (gateway, wallet, auth, transfer, notification, webhook, devca, chaos)
db/            per-service SQL migrations and sqlc query definitions
deploy/        Docker Compose Postgres init, Prometheus, Grafana, Kind, Helm charts, and Toxiproxy config
gen/           generated Protobuf and gRPC Go stubs
internal/      domain logic, sqlc stores, and shared platform packages (authn, grpcx, idempotency, mtls, outbox, telemetry)
loadtest/      k6 load-testing scripts
proto/         Protobuf service contracts managed with buf
scripts/       helper scripts for local traffic and verification
```

## Quick Start

Requirements: Go `1.26+`, Docker, and `make`. (`golang-migrate`, `sqlc`, and `buf` CLIs are only needed when modifying migrations, SQL queries, or `.proto` contracts.)

```bash
make keys certs
docker compose --profile app up -d --build --wait
```

Send requests using the `.http` collections in [`api/`](api), starting with [`api/gateway.http`](api/gateway.http).

To run a single service from source while infrastructure runs in Docker:

```bash
docker compose up -d
make run-wallet   # or run-auth, run-transfer, run-notification, run-gateway, run-webhook
```

## Testing & Verification

```bash
make test             # fast unit tests
make test-integration # integration tests against ephemeral PostgreSQL, Redis, and NATS containers
make test-race        # full suite with the Go race detector
make loadtest         # k6 transfer load test against the running compose stack
```

## License

[MIT](LICENSE)
