# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

See [AGENTS.md](AGENTS.md) for the canonical agent command reference; the most useful commands are duplicated below.

## Repository layout

Yarn (Berry) monorepo. Workspaces live under `packages/*` and share `tsconfig.base.json` via TypeScript project references (`composite: true`). Strict TS is enabled globally, including `noUncheckedIndexedAccess`.

- **backend-lib** — the core. Holds nearly all business logic, the Drizzle schema, ClickHouse access, Temporal workflows/activities, messaging, and config. Every other server package depends on it.
- **isomorphic-lib** — pure types/utilities shared between backend and frontend (no Node-only deps). TypeBox schemas and shared constants live here.
- **api** — Fastify HTTP server (TypeBox type-provider). Entry: `scripts/startServer.ts` → `src/buildApp`.
- **worker** — Temporal worker process that runs workflows/activities. Entry: `scripts/startWorker.ts` → `src/buildWorker`.
- **dashboard** — Next.js frontend (MUI, Zustand, TanStack Query/Table, `@xyflow/react` for the journey builder).
- **lite** — single-process bundle that runs the API, worker, and dashboard together (used for simple self-hosted deploys). Entry: `scripts/startLite.ts`.
- **admin-cli** — operational CLI (bootstrap, migrations, maintenance jobs). Run via `yarn admin <command>`.
- **emailo** — low-code email editor (ProseMirror/Slate based), embedded in the dashboard.
- **docs** — Mintlify documentation site.

## Architecture (the big picture)

Dittofeed is a customer-engagement platform: ingest user events → compute segments/user-properties → run journeys/broadcasts → send messages across channels.

Four backing stores, each with a distinct role — know which one you're touching:
- **Postgres** (via **Drizzle ORM**) — relational config/state: workspaces, journeys, segments, templates, subscription groups, etc. Schema in `packages/backend-lib/src/db/schema.ts`; migrations in `packages/backend-lib/drizzle/` (drizzle-kit). The repo migrated off Prisma — prefer Drizzle for all new DB code.
- **ClickHouse** — high-volume event data and computed-property/segment assignment storage (analytics, deliveries). Access in `packages/backend-lib/src/clickhouse.ts` and `computedProperties/`.
- **Temporal** — durable workflow orchestration. The two central workflow families:
  - `journeys/userWorkflow` — one workflow execution per user per journey, driving them through journey nodes.
  - `computedProperties/computePropertiesWorkflow` — recomputes segment membership and user properties incrementally on a schedule.
- **Kafka** — event ingestion buffer (optional in lite/dev).

`packages/backend-lib/src/config.ts` resolves almost all environment variables and configuration — start there when tracing how a setting flows through the system.

Messaging channels (email, SMS, push, webhook, Slack) live under `packages/backend-lib/src/destinations/` and `messaging.ts`, with provider SDKs (SendGrid, SES, Resend, Postmark, Twilio, etc.). Templates render via LiquidJS and MJML.

## Common commands

Per-package scripts run through Yarn workspaces. Substitute the package name as needed.

```bash
# Type-check a package (tsc --build, respecting project references)
yarn workspace backend-lib check

# Lint / autofix a single file
yarn workspace backend-lib eslint src/resources.test.ts --fix

# Run a single test file
yarn jest packages/backend-lib/src/resources.test.ts

# Filter by test name within a file
yarn jest packages/backend-lib/src/resources.test.ts -t "specific test name"

# Run a test and tee output to a timestamped file in .tmp/ (preferred for large tests
# to avoid flooding context — then search the file with Read/Grep)
yarn test:file packages/backend-lib/src/resources.test.ts

# More verbose logs during a test
LOG_LEVEL=debug yarn jest packages/backend-lib/src/resources.test.ts

# Temporal-tagged backend tests only
yarn test:temporal

# Run a service in dev
yarn workspace api dev
yarn workspace worker dev
yarn workspace dashboard dev
yarn workspace lite dev

# Operational CLI (bootstrap, migrations, maintenance)
yarn admin bootstrap
```

Jest is configured with `jest-runner-groups` and per-package projects (`backend-lib`, `backend-lib-jsdom`, `dashboard`, `api`, `emailo`). Use `--group=<name>` to scope by group tag.

## Local infrastructure

`docker-compose.yaml` brings up the full dev stack (postgres, clickhouse, temporal + temporal-ui, kafka, otel-collector/grafana/prometheus/zipkin, mail-server, blob-storage). `docker-compose.lite.yaml` runs the single-process `lite` variant; `docker-compose.ee.yaml` is the enterprise build. `.tmp/` is the scratch directory for disposable debug output.

## Conventions

- **Error handling**: `neverthrow` `Result`/`ResultAsync` is used pervasively for fallible operations rather than throwing — match the surrounding style.
- **Validation & types**: runtime schemas use TypeBox (`@sinclair/typebox`); shared type definitions belong in `isomorphic-lib`, not duplicated per package.
- **ESLint**: airbnb-typescript + prettier + `simple-import-sort`. Run `lint`/`check` before considering work done.
