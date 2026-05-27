# Running the service locally

This walks through running `meet-common-settings` against a self-contained RabbitMQ + PostgreSQL stack on your machine, seeding a few users, and publishing a test message to watch the full path end-to-end. Useful for smoke-testing a change before pushing.

The full Meet stack is not required. We reproduce just the `meet_user` table from Meet's `0001_initial.py` migration — that's everything this service touches.

## Prerequisites

- Docker (the compose plugin works fine — Docker 20.10+).
- Node.js 20+.
- `psql` is handy but not required (you can `docker exec` into the container).

## Set up the dependency stack

These files live wherever you like; the examples below put them under `/tmp/meet-e2e/`. Nothing in this section gets committed.

### `compose.test.yml`

```yaml
name: meet-cs-e2e
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: meet
      POSTGRES_USER: meet
      POSTGRES_PASSWORD: meet
    ports:
      - "5433:5432"
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/01-init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U meet -d meet"]
      interval: 1s
      timeout: 3s
      retries: 30

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5673:5672"
      - "15673:15672"
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 2s
      timeout: 5s
      retries: 30
```

Ports are intentionally non-default (`5433`, `5673`, `15673`) so they don't collide with anything else you might already have on `5432`/`5672`.

### `init.sql`

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE meet_user (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    password VARCHAR(128) NOT NULL DEFAULT '!unusable',
    last_login TIMESTAMPTZ,
    is_superuser BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    sub VARCHAR(255) UNIQUE,
    email VARCHAR(254),
    admin_email VARCHAR(254) UNIQUE,
    language VARCHAR(10) NOT NULL DEFAULT 'en-us',
    timezone VARCHAR(63) NOT NULL DEFAULT 'UTC',
    is_device BOOLEAN NOT NULL DEFAULT FALSE,
    is_staff BOOLEAN NOT NULL DEFAULT FALSE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE
);

INSERT INTO meet_user (sub, email, language, timezone, updated_at) VALUES
  ('oidc-sub-alice', 'Alice@example.com', 'en-us', 'UTC',              NOW() - INTERVAL '1 hour'),
  ('oidc-sub-bob',   'bob@example.com',   'fr-fr', 'Europe/Paris',     NOW() - INTERVAL '1 hour'),
  ('oidc-sub-carol', 'carol@example.com', 'en-us', 'America/New_York', NOW() - INTERVAL '1 hour');
```

The schema matches Meet's `0001_initial.py` migration for the subset of columns relevant here. Three seeded users with distinct starting states are enough to cover the happy paths.

### Bring it up

```sh
cd /tmp/meet-e2e
docker compose -f compose.test.yml up -d --wait
```

Both containers should report `Healthy`. If not, `docker compose -f compose.test.yml logs` is your friend.

Sanity-check the seeded data:

```sh
docker exec meet-cs-e2e-postgres-1 \
  psql -U meet -d meet -c "SELECT email, language, timezone FROM meet_user ORDER BY email;"
```

RabbitMQ management UI is at http://localhost:15673 (`guest` / `guest`).

## Run the service

From the `meet-common-settings` checkout:

```sh
RABBITMQ_URL=amqp://guest:guest@localhost:5673 \
DATABASE_URL=postgres://meet:meet@localhost:5433/meet \
HEALTH_PORT=8090 \
LOG_LEVEL=info \
npm run dev
```

You should see, within a second or two:

```
"health server listening" port=8090
"connecting to RabbitMQ" exchange=settings routingKey=user.settings.updated queue=meet.user_settings prefetch=1
"Connected to server"
"Confirm channel created"
"Channel prefetch set" prefetch=1
"Subscribed to queue"
"consumer subscribed and processing messages"
```

Probe the HTTP endpoints from another terminal:

```sh
curl -s -o /dev/null -w "healthz: %{http_code}\n" http://localhost:8090/healthz
curl -s -o /dev/null -w "readyz: %{http_code}\n"  http://localhost:8090/readyz
curl -s http://localhost:8090/metrics | grep mcs_
```

`/readyz` should be `200`; `/metrics` should show all the `mcs_*` counters at zero.

## Publish a test message

A one-file Node script (anywhere you like):

```js
// publish.mjs
import { RabbitMQClient } from '<absolute-path-to>/meet-common-settings/node_modules/@linagora/rabbitmq-client/dist/index.js';

const client = new RabbitMQClient({ url: 'amqp://guest:guest@localhost:5673' });
await client.init();

await client.publish('settings', 'user.settings.updated', {
  source: 'local-test',
  nickname: 'alice',
  request_id: `local-${Date.now()}`,
  timestamp: Date.now(),
  version: 1,
  payload: {
    email: 'alice@example.com',
    language: 'fr',
    timezone: 'Europe/Berlin',
  },
});

await client.close();
```

Run it: `node publish.mjs`.

In the service log you should see something like:

```
"user settings updated" requestId=local-... emailHash=ff8d... rowCount=1
                        languageUpdated=true timezoneUpdated=true latencyMs=4
```

And in Postgres:

```sh
docker exec meet-cs-e2e-postgres-1 \
  psql -U meet -d meet -c "SELECT email, language, timezone, updated_at FROM meet_user WHERE email ILIKE 'alice@example.com';"
```

Alice's row should now show `fr-fr` / `Europe/Berlin` with a fresh `updated_at`.

## Scenarios worth exercising

Vary the payload to walk every outcome label. Each row corresponds to a unique `outcome` you'll see in the log and in `mcs_messages_processed_total`.

| Payload | Expected outcome |
|---|---|
| `{ email: 'alice@example.com', language: 'fr', timezone: 'Europe/Berlin' }` | `updated` (both fields) |
| `{ email: 'bob@example.com', language: 'en' }` | `updated` (language only) |
| `{ email: 'carol@example.com', timezone: 'Asia/Tokyo' }` | `updated` (timezone only) |
| `{ email: 'ghost@example.com', language: 'de' }` | `unknown_user` (no row matches) |
| `{ email: 'alice@example.com', language: 'es' }` | `no_syncable_fields` (Spanish has no Django mapping; no timezone provided) |
| `{ language: 'fr' }` (no email) | `no_email` |
| `{ foo: 'bar' }` (no payload) | `invalid_payload` |

The `language` codes the service understands by default are `en`, `fr`, `nl`, `de` (each maps to the Django `XX-XX` form). Anything else is dropped unless you set `LANGUAGE_MAP_OVERRIDES`.

## Tear it all down

```sh
# Stop the npm run dev process (Ctrl-C or kill the shell)
docker compose -f /tmp/meet-e2e/compose.test.yml down -v
rm -rf /tmp/meet-e2e
```

`down -v` removes the test data volume, which is what you want — next time you bring it up, the init SQL runs again from scratch.
