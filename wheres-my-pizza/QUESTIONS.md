## Architecture (Hexagon / Ports & Adapters)

### Services are split into inbound/outbound ports and pure domain packages
- [ ] Yes
- [ ] No

### `api-gateway`, `kitchen`, `delivery` are built as **three independent Go binaries**
- [ ] Yes
- [ ] No

### No import cycles: upper layers never import lower ones
- [ ] Yes
- [ ] No

### All third-party tech (`pgx`, `amqp091-go`) are wrapped in adapters
- [ ] Yes
- [ ] No

## PostgreSQL & Transactional Outbox

### `init.sql` creates every table, enum `order_status`, indexes
- [ ] Yes
- [ ] No

### Order insert **and** first outbox event happen in one SQL transaction
- [ ] Yes
- [ ] No

### Unpublished events are selected with `FOR UPDATE SKIP LOCKED`
- [ ] Yes
- [ ] No

### `published_at` is set only after broker **ACK**
- [ ] Yes
- [ ] No

### Inbox tables use `PRIMARY KEY(event_id)` + `ON CONFLICT DO NOTHING`
- [ ] Yes
- [ ] No

## RabbitMQ – Topology, QoS, Retry, DLQ

### Exchanges / queues exactly match the declared topology
- [ ] Yes
- [ ] No

### Prefetch is applied via `basic.Qos(<RMQ_PREFETCH>, 0, true)`
- [ ] Yes
- [ ] No

### NACK → retry → TTL → source-exchange path works as specified
- [ ] Yes
- [ ] No

### Separate retry exchange **or** routing-key namespace for commands vs events
- [ ] Yes
- [ ] No

### After max retries the message is routed to `orders.dlq.q`
- [ ] Yes
- [ ] No

### All publishes use **Publisher Confirms**
- [ ] Yes
- [ ] No

## HTTP API

### `POST /orders` and `GET /orders/{id}` implemented
- [ ] Yes
- [ ] No

### Error responses are JSON with `"error"` field
- [ ] Yes
- [ ] No

### Query param `force_payment_fail` accepted **only in DEV**
- [ ] Yes
- [ ] No

### API never leaks sensitive data (PII) in payloads
- [ ] Yes
- [ ] No

## Configuration / Deployment

### All settings come from **environment variables**; only `--help` flag exists
- [ ] Yes
- [ ] No

### `docker compose up` boots everything; app works end-to-end
- [ ] Yes
- [ ] No

### Services handle `SIGINT`/`SIGTERM`, drain consumers and finish gracefully
- [ ] Yes
- [ ] No

### Startup errors exit with non-zero code and clear message
- [ ] Yes
- [ ] No

## Logging & Observability

### `log/slog` used with structured fields (`order_id`, `event`)
- [ ] Yes
- [ ] No

### Log levels: **Info** on publish  /  **Warn** on retry  /  **Error** on DLQ
- [ ] Yes
- [ ] No

### Prometheus/expvar metrics expose DLQ rate and publish latency
- [ ] Yes
- [ ] No

## Testing

### Unit tests cover pure business logic (no RabbitMQ / PG)
- [ ] Yes
- [ ] No

### Contract tests validate JSON schema of events
- [ ] Yes
- [ ] No

### End-to-end tests (Testcontainers-go) cover all 5 required scenarios
- [ ] Yes
- [ ] No

### Retry TTL is overridden ≤ 100 ms; full `go test ./...` finishes < 30 s
- [ ] Yes
- [ ] No

## Code Quality

### Passes `gofumpt`; uses only stdlib + `amqp091-go` + `pgx`
- [ ] Yes
- [ ] No

### No cyclic imports or unnecessary global state
- [ ] Yes
- [ ] No

## Documentation

### README contains the provided **Mermaid** topology diagram
- [ ] Yes
- [ ] No

### Steps to reproduce locally are clear and complete
- [ ] Yes
- [ ] No


## Detailed Feedback

### What was great? What you liked the most about the program and the team performance?

### What could be better? How those improvements could positively impact the outcome?