# Architecture

## What this service is for

Twake Workplace has a central **common-settings** service where users edit their language, timezone, display name, avatar, and other profile fields. When a user saves a change there, common-settings publishes a RabbitMQ message so every other application in the suite can react.

Meet (the LiveKit-based video conferencing app) is one of those applications. It stores its own copy of a user's language and timezone in PostgreSQL. Until now, those values only got refreshed when the user logged into Meet again, because Meet pulls them from OIDC userinfo claims at sign-in time. That meant a user could change their language in common-settings and still see the old language in Meet until the next login.

`meet-common-settings` closes that gap. It is a small sidecar that listens to the common-settings exchange and writes the changed fields into Meet's database in near real time. Meet itself is untouched.

## How it fits together

```
┌────────────────────┐    AMQP (topic)    ┌────────────────────────┐    SQL    ┌────────────┐
│  common-settings   │ ─────────────────► │  meet-common-settings  │ ────────► │  Meet PG   │
│   (publisher)      │  settings          │   (this service)       │  UPDATE   │  meet_user │
│                    │  user.settings     │   1 replica            │  by email │            │
│                    │  .updated          │                        │           │            │
└────────────────────┘                    └────────────────────────┘           └────────────┘
```

The service has exactly two outbound connections: RabbitMQ and PostgreSQL. It exposes an HTTP port on `/healthz`, `/readyz`, and `/metrics` for probes and metrics scrapers.

## Why direct database UPDATEs

We considered three approaches:

1. **Direct PostgreSQL UPDATE.** This is what we ship.
2. **A new service-to-service admin API on Meet** that the sidecar calls over HTTP.
3. **Embedding the consumer inside Meet's Django backend** as a management command or Celery worker.

Option 2 would have meant changing both repos: adding an authenticated admin endpoint to Meet (today the user-update endpoint is gated by `IsSelf`, so it can only be called by the user themselves), then having the sidecar call it. That's a much bigger surface area for a write that only ever touches two columns.

Option 3 would have coupled Meet's deployment to RabbitMQ, when today its Celery broker is Redis. Adding a new messaging dependency to the main backend felt heavier than necessary.

Option 1 keeps Meet completely untouched. The cost is that the sidecar has to know the table name and column names of Meet's user table. We mitigate that by:

- Granting the sidecar's PostgreSQL role only `SELECT (email), UPDATE (language, timezone, updated_at)` on `meet_user`. The blast radius of a schema mistake is bounded.
- Making the table name configurable via `MEET_USER_TABLE` so a Django rename can be absorbed without a code change.
- Validating the identifier against an allowlist regex before interpolating it into SQL.

## How users are matched

Common-settings identifies users by `nickname` (a username string). Meet identifies users by `sub` (their OIDC subject claim). There is no direct mapping between the two: `nickname` is not Meet's `sub`.

The one identifier shared by both systems is **email**. Meet's prod deployment already has `OIDC_FALLBACK_TO_EMAIL_FOR_IDENTIFICATION=True`, meaning Meet itself uses email as a fallback when sub doesn't resolve. We follow the same convention: `WHERE email ILIKE $3`.

If a settings message arrives for a user who has never logged into Meet (and therefore has no row in `meet_user`), the UPDATE returns zero rows. We log an info-level event and ack the message. The user's settings will be applied automatically on their first OIDC sign-in, because Meet pulls language and timezone from OIDC userinfo claims at that point.

## Which fields we sync

| Common-settings payload | Meet column | Notes |
|---|---|---|
| `language` (ISO 639-1, e.g. `"en"`) | `meet_user.language` | Mapped to Django's `LANGUAGES` codes (`en-us`, `fr-fr`, `nl-nl`, `de-de`). Unsupported codes are dropped, not raised. |
| `timezone` (IANA, e.g. `"Europe/Paris"`) | `meet_user.timezone` | Passed through unchanged. |
| `avatar` | — | Ignored. Meet has no avatar field; adding one is a separate, larger project. |
| `display_name` | — | Ignored. Meet's `full_name` and `short_name` are populated from OIDC userinfo on every login; writing them here would race with the OIDC backend. |
| Everything else | — | Ignored. |

## Idempotency and ordering

We do not track per-user state. The handler runs a single `UPDATE ... WHERE email ILIKE $1` and acks. Re-applying the same payload is a no-op (the values are the same). Applying an out-of-order older payload would temporarily revert settings until the next real update — but in practice this doesn't happen because:

- The service is meant to run as **a single process**: never more than one consumer attached to the queue.
- RabbitMQ's channel prefetch is set to **1**, so the broker dispatches one message at a time and waits for the ack before sending the next.
- A single durable queue with a single consumer preserves message order.

That gives us strict serial processing without any application-side state.

## Failure modes and what they mean

The handler classifies every outcome into one of seven labels, all reported as `mcs_messages_processed_total{outcome=...}`:

| Outcome | What it means | Ack? |
|---|---|---|
| `updated` | Found the user, applied the change. | Yes |
| `unknown_user` | No row matched the email. User probably hasn't logged into Meet yet. | Yes |
| `no_email` | Message had no email field. Can't match anyone. | Yes |
| `no_syncable_fields` | Message had no language or timezone (or only an unsupported language code). | Yes |
| `invalid_payload` | JSON parsed but failed schema validation. | Yes (poison) |
| `db_error` | Postgres returned a transient error (connection refused, deadlock, etc.). | Throw → library retries with backoff, then DLQs |
| `unexpected_error` | Postgres returned a permanent error (column missing, etc.). | Yes |

The "throw → retry → DLQ" path is provided by `@linagora/rabbitmq-client`: it catches the thrown error, retries up to `RABBITMQ_MAX_RETRIES` times with `RABBITMQ_RETRY_DELAY` ms between attempts, then nacks to the dead-letter queue. The library also has reconnection logic for total broker outages.

## Why a sidecar instead of merging into common-settings

Each downstream app has its own data model and its own way of representing users. If common-settings carried the fan-out responsibility, it would need to know how to talk to every other app's database, OIDC mapping, validation rules, and schema. Keeping the consumer adjacent to the app it writes to keeps that knowledge in one place. The pattern is repeatable: future apps can copy this service's shape and adapt the handler.
