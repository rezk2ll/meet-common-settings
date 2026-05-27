# Operations

This document covers what an operator needs to know: how to provision the service's dependencies, what to watch, and what to do when something looks wrong. Deployment topology (containers, Kubernetes, Helm) is out of scope — that lives with the chart.

## Database role

The PostgreSQL role embedded in `DATABASE_URL` should be granted only what the consumer needs and nothing more:

```sql
CREATE ROLE meet_common_settings WITH LOGIN PASSWORD '...';
GRANT CONNECT ON DATABASE meet TO meet_common_settings;
GRANT USAGE ON SCHEMA public TO meet_common_settings;
GRANT SELECT (email), UPDATE (language, timezone, updated_at) ON meet_user TO meet_common_settings;
```

`INSERT`, `DELETE`, and writes to any other table or column are not granted. If a future change to the service tries to write something else, Postgres will reject it rather than silently corrupting data.

## RabbitMQ permissions

The user in `RABBITMQ_URL` needs:

- `read` on the `settings` exchange.
- `read`, `write`, and `configure` on the `meet.user_settings` queue (the service declares the queue on startup).
- `read` and `write` on the dead-letter exchange and queue that `@linagora/rabbitmq-client` auto-declares (the names follow the library's convention; check its docs if you provision RabbitMQ permissions via a separate tool).

## What to monitor

### Liveness and readiness

- `GET /healthz` — process is alive. Suitable for a basic restart probe.
- `GET /readyz` — the consumer is subscribed AND the broker connection is up AND PostgreSQL responds to `SELECT 1`. Returns 503 with a reason during reconnects, schema problems, or database outages.

If `/readyz` flips to 503 for more than a minute or two, the service is not consuming messages and the queue is filling up. The reason field in the response points at the broken dependency.

### Prometheus metrics

The service exposes these on `/metrics`:

- `mcs_messages_processed_total{outcome}` — counter, one increment per processed message.
- `mcs_message_latency_seconds{outcome}` — histogram of per-message wall time including the database call.
- `mcs_db_errors_total` — counter, increments on any database exception (transient or permanent).
- Plus the default Node.js process metrics (heap, event loop lag, GC).

Suggested alerts:

| Alert | Condition | What it tells you |
|---|---|---|
| Service unreachable | `up{job="meet-common-settings"} == 0 for 5m` | The process is down or Prometheus can't scrape. |
| Permanent errors rising | `rate(mcs_messages_processed_total{outcome="unexpected_error"}[5m]) > 0` | Likely a schema drift (a column was renamed or dropped). Investigate immediately. |
| Queue is backing up | A `rabbitmq_queue_messages{queue="meet.user_settings"}` alert > N for X minutes | Either the consumer is slow or `/readyz` is down. Check the readiness probe reason. |
| DLQ growing | `rate(rabbitmq_queue_messages_published_total{queue=~"meet.user_settings.dlq"}[15m]) > 0` | Messages are exhausting their retries. Means a sustained DB outage or a poison-message pattern. |

### Logs

Every message produces exactly one structured log line in pino's JSON format. Fields you can index on:

- `requestId` — the upstream common-settings request_id, useful for cross-service tracing.
- `version` — the payload version field.
- `emailHash` — the first 16 hex chars of `sha256(lowercase(email))`. Lets you correlate without storing PII in your log aggregator.
- `outcome` — same labels as the metric.
- `latencyMs` — wall time end-to-end.

The library also emits its own logs through the same pino instance: connection events, retries, DLQ routings.

## Common failure modes

### "I changed my language in common-settings but Meet still shows the old one"

In order of likelihood:

1. **Browser cache.** Meet's frontend caches language. Reload.
2. **The user has not logged into Meet yet.** Without a `meet_user` row, the UPDATE matches zero rows. Log line: `"no Meet user matched; skipping"`. The user's settings will apply on first login.
3. **The language code in common-settings is one we don't map.** Look for `"language code has no Django mapping; skipping language update"`. Meet currently only supports `en-us`, `fr-fr`, `nl-nl`, `de-de`. To add another, override `LANGUAGE_MAP_OVERRIDES` (see [configuration](#configuration)) or add it to `src/language.ts`.
4. **The service is not consuming.** Check `/readyz` and the broker UI's consumer count for `meet.user_settings`.

### Postgres is down

Each message in flight is retried up to `RABBITMQ_MAX_RETRIES` (default 5) times with a `RABBITMQ_RETRY_DELAY` ms (default 1000) wait between attempts. After exhausting retries, the message goes to the DLQ. Subsequent messages keep flowing into the queue.

For brief outages (under a few seconds) this is fine — the in-flight message is retried and succeeds. For longer outages, you have two options:

- **Let it DLQ.** Once Postgres is back up, replay the DLQ. The `@linagora/rabbitmq-client` docs cover how the DLQ is named and how to drain it.
- **Stop the consumer process** before retries exhaust. The queue will fill but no messages will be lost. Restart when Postgres is healthy.

### A column was renamed in Meet

You will see `mcs_messages_processed_total{outcome="unexpected_error"}` climb sharply, every message logs `"permanent database error; acking to avoid poison-message loop"` with Postgres error code `42703`. The service acks the messages (so they're not retried in a loop), and the data is silently lost until you ship a fix.

Fix path:

- Roll back to the previous image while you update `src/db.ts` to match the new column name (or update the SQL).
- Release a new image, redeploy.
- If you have the DLQ wired into something durable, you can also replay the lost window from there.

### Broker outage

`@linagora/rabbitmq-client` reconnects automatically and restores subscriptions. During the outage, `/readyz` returns 503 with reason `consumer_not_connected`. After reconnect, it returns to 200 within a few seconds. No special intervention needed.

If the outage is permanent (broker decommissioned, URL changed), update `RABBITMQ_URL` and restart the process.

## Configuration

All configuration is via environment variables. Defaults are listed in [`.env.example`](../.env.example).

| Variable | Required | Default | What it does |
|---|---|---|---|
| `RABBITMQ_URL` | yes | — | AMQP DSN. |
| `RABBITMQ_EXCHANGE` | no | `settings` | Topic exchange to bind to. |
| `RABBITMQ_ROUTING_KEY` | no | `user.settings.updated` | Binding key. |
| `RABBITMQ_QUEUE` | no | `meet.user_settings` | Consumer queue name. |
| `RABBITMQ_PREFETCH` | no | `1` | QoS prefetch. Keep at 1 to preserve ordering. |
| `RABBITMQ_MAX_RETRIES` | no | `5` | Handler retries before the message is sent to the DLQ. |
| `RABBITMQ_RETRY_DELAY` | no | `1000` | Delay between handler retries, in ms. |
| `DATABASE_URL` | yes | — | PostgreSQL DSN for the Meet database. |
| `MEET_USER_TABLE` | no | `meet_user` | User table override, in case Django renames it. |
| `LANGUAGE_MAP_OVERRIDES` | no | `{}` | JSON map of additional ISO-639-1 → Django language codes. Example: `{"es":"fr-fr"}`. |
| `LOG_LEVEL` | no | `info` | pino level: `trace`, `debug`, `info`, `warn`, `error`, `fatal`. |
| `HEALTH_PORT` | no | `8080` | Port for `/healthz`, `/readyz`, `/metrics`. |
| `SHUTDOWN_TIMEOUT_MS` | no | `10000` | Grace period on SIGTERM. The broker client uses the same value as its `closeTimeout`, so this is how long we'll wait for in-flight handlers to finish before forcing exit. |

## Restarts and single-consumer invariant

The ordering guarantee relies on at most one consumer being attached to the queue at any time. Whatever runs this service must enforce that: do not scale to multiple replicas, and stop the old process before starting a new one during upgrades. Brief downtime is harmless — messages accumulate in the queue and drain when the new process is up.
