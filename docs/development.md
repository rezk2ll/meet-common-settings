# Development

## Prerequisites

- Node.js 20 or newer.
- Docker (only for the integration tests, which spin up real Postgres and RabbitMQ containers).

## Setup

```sh
npm install
cp .env.example .env  # then edit DATABASE_URL and RABBITMQ_URL
npm run dev           # tsx watch
```

For a development RabbitMQ + Postgres, the easiest option is the docker-compose file from the [common-settings](https://github.com/linagora/twake-workplace-common-settings) repo. It already exposes the broker on `amqp://guest:guest@localhost:5672` and provisions the `settings` exchange. Point your local `DATABASE_URL` at a throwaway Postgres with a fixture `meet_user` table — the integration tests show the minimal schema.

## Project layout

```
src/
  index.ts        Process entrypoint: load config, wire deps, run consumer + health server, handle SIGTERM
  config.ts       Env-var parsing with Zod
  consumer.ts     RabbitMQ subscribe loop, ack/throw semantics
  handler.ts      Per-message logic. The interesting part.
  db.ts           pg pool wrapper, single UPDATE statement
  schema.ts       Zod schema for the message envelope and payload
  language.ts     ISO 639-1 → Django LANGUAGES mapping
  metrics.ts      prom-client registry and counters
  logger.ts       pino instance plus email hashing helper
  health.ts       HTTP server for /healthz, /readyz, /metrics

tests/
  unit/           Fast, no docker. Pure-function tests with mocked db.
  integration/   testcontainers spin up Postgres; verifies SQL behaviour.
```

The shape is deliberately flat. Every file has one job and the call graph is shallow. If you find yourself adding a sixth or seventh kind of dependency, the abstraction is probably wrong.

## Testing

```sh
npm run test:unit         # fast, no docker
npm run test:integration  # requires docker (testcontainers)
npm test                  # both
```

The unit tests cover the handler exhaustively — every outcome label has at least one test, including the error-classification branches. Integration tests verify the actual SQL runs against a real Postgres image.

When changing behavior:

- Add a unit test for the new outcome.
- If the change touches the SQL, add an integration test that exercises it.
- Run `npm test` locally before pushing. CI runs both.

## Linting and types

```sh
npm run lint        # eslint
npm run format      # prettier --write
npm run typecheck   # tsc --noEmit
```

The pre-merge gate is: typecheck clean, lint clean, unit tests pass. Integration tests run on every push but are not blocking by default (they need Docker in CI).

## Building the image

```sh
docker build -t meet-common-settings:dev .
```

The Dockerfile is a two-stage build. The runtime is `gcr.io/distroless/nodejs20-debian12:nonroot` — no shell, no package manager, no root user. If you need to debug a running container, you can swap the base image temporarily to `node:20-bookworm-slim` for that build.

## Releasing

1. Bump the version in `package.json` and add a line to `CHANGELOG.md`.
2. Commit and tag: `git tag vX.Y.Z && git push origin vX.Y.Z`.
3. Push the image to the registry (CI handles this on tag push if configured).

Updating the deployment to pick up the new image is handled separately by whichever tool owns the deployment.

## Adding a new synced field

The shortest path:

1. Add the field to the Zod schema in `src/schema.ts`.
2. Add a column to the `UPDATE` in `src/db.ts` and to the `UserSettingsUpdate` type. Wrap it in `COALESCE($n, column)` so omitting it leaves the existing value alone.
3. Extend `handler.ts` to copy the field from `payload` into `updates`, with any validation or mapping you need.
4. Add tests in `tests/unit/handler.spec.ts` and `tests/integration/db.spec.ts`.
5. Update the architecture doc's "Which fields we sync" table.
6. Add the column to the PostgreSQL grant in the [operations](operations.md#database-role) doc and in production.

Don't ship a new field without granting it. The role is least-privilege by design, so the UPDATE will fail loudly rather than silently drop the column from the write.
