# meet-common-settings

Sidecar service that consumes user settings update messages from the Twake Workplace common-settings RabbitMQ exchange and applies the changes to Meet's PostgreSQL database.

## What it does

- Subscribes to the `settings` topic exchange on RabbitMQ with routing key `user.settings.updated`.
- For each message, matches the user by `payload.email` (case-insensitive) and runs:
  ```sql
  UPDATE meet_user
     SET language = COALESCE($1, language),
         timezone = COALESCE($2, timezone),
         updated_at = NOW()
   WHERE email ILIKE $3
  ```
- Only `language` and `timezone` are synced. `language` is mapped from ISO 639-1 (`en`, `fr`, ...) to Django's `LANGUAGES` codes (`en-us`, `fr-fr`, `nl-nl`, `de-de`).
- Unknown users (never logged into Meet) are skipped silently.
- The service holds no state. Re-applying the same payload is a no-op; the queue's natural ordering plus a single consumer make replays safe.

## Configuration

All configuration is via environment variables. See `.env.example` for defaults.

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `RABBITMQ_URL` | yes | — | AMQP DSN |
| `RABBITMQ_EXCHANGE` | no | `settings` | Topic exchange to bind to |
| `RABBITMQ_ROUTING_KEY` | no | `user.settings.updated` | Binding key |
| `RABBITMQ_QUEUE` | no | `meet.user_settings` | Consumer queue name |
| `RABBITMQ_PREFETCH` | no | `1` | QoS prefetch count |
| `DATABASE_URL` | yes | — | PostgreSQL DSN for the Meet database |
| `MEET_USER_TABLE` | no | `meet_user` | User table override (defensive) |
| `LANGUAGE_MAP_OVERRIDES` | no | `{}` | JSON map of additional ISO → Django language codes |
| `LOG_LEVEL` | no | `info` | pino log level |
| `HEALTH_PORT` | no | `8080` | HTTP port for probes and metrics |
| `SHUTDOWN_TIMEOUT_MS` | no | `10000` | Grace period on SIGTERM |

The Postgres role used by the service should be granted only `SELECT, UPDATE (language, timezone, updated_at) ON meet_user`. No `INSERT` or `DELETE` is performed.

## HTTP endpoints

| Path | Purpose |
|---|---|
| `GET /healthz` | Liveness: returns 200 while the process is alive |
| `GET /readyz` | Readiness: 200 once the consumer is connected and the database responds to `SELECT 1` |
| `GET /metrics` | Prometheus metrics (process metrics + `mcs_messages_processed_total{outcome=...}`, `mcs_message_latency_seconds`, `mcs_db_errors_total`) |

## Development

```sh
npm install
cp .env.example .env  # then edit
npm run dev
```

Tests:

```sh
npm run test:unit         # fast, no docker
npm run test:integration  # requires docker (testcontainers)
npm test                  # both
```

## Further reading

- [docs/architecture.md](docs/architecture.md) — what the service does, message flow, design rationale.
- [docs/operations.md](docs/operations.md) — provisioning dependencies, monitoring, alerting, troubleshooting.
- [docs/development.md](docs/development.md) — local setup, tests, releasing.
- [docs/running-locally.md](docs/running-locally.md) — end-to-end smoke test against Docker postgres + rabbitmq.
