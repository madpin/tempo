# Tempo — First Technical Decisions (Webapp)

*Companion to `PRD.md`. Records the initial stack choices for the **webapp** — Tempo's first surface — plus the local development setup and the dev/qa/prod environment story. Mobile and desktop are deferred to later documents.*

---

## 0. Scope

This document covers the **webapp only**:

- Next.js application (canonical UI + server-side API).
- Postgres as the only data store for v1.
- Local development via `docker-compose` against a Postgres container on a non-default port (the contributor's machine may already have a Postgres running on 5432).
- Three target environments: **dev** (local), **qa** (shared remote), **prod** (defined later).

Mobile (React Native), desktop (Tauri), the offline sync layer, and the cross-platform engine package live in separate documents. The webapp is the one surface where v1 must be excellent.

---

## 1. Guiding Principles

1. **Boring where possible, novel where it matters.** Postgres is boring. The constraint engine and AI authoring layer are where Tempo earns its keep — concentrate complexity there.
2. **Optimize for fast iteration in v1.** The hard problems (engine quality, prompt design, UX) sit upstream of throughput. Don't pre-optimize for scale we don't have.
3. **One language across server and client.** The same scheduling logic must run on both sides; pick a stack that makes that natural rather than bolted-on.
4. **Configuration is explicit, secrets are external.** Each environment has its own committed config file describing *shape*; secrets are injected at deploy time and never live in the repo.
5. **Pluggable AI provider.** The LLM is a strategic dependency — wrap it behind a thin internal interface from day one.

---

## 2. Stack at a Glance

| Layer | Choice | Why (one line) |
|-------|--------|----------------|
| **Language** | TypeScript (strict) | Same types on server and client; mature ecosystem |
| **Runtime** | Node.js 22 LTS | Long support window; first-class Next.js support |
| **Framework** | Next.js 15 (App Router) + React 19 | Server components, server actions, edge-friendly, mature |
| **API style** | Server Actions for in-app calls; tRPC for typed RPC; REST for external webhooks/integrations | End-to-end typed; minimal schema duplication |
| **Database** | PostgreSQL 16 | Relational shape fits the domain; JSONB for flexible config; mature tooling |
| **ORM / migrations** | Drizzle ORM + Drizzle Kit | Type-safe, SQL-first, lightweight migrations |
| **Auth** | Clerk (managed) | Drop-in OAuth (Google + Microsoft, which we already need for calendar sync) and sessions |
| **Background jobs** | Inngest | Replans, reminders, scheduled calendar polling — without standing up a queue ourselves |
| **LLM** | Anthropic Claude via the official SDK; behind an internal `AIProvider` interface | Strong structured output, long context, good tool-use story |
| **External calendar sync** | Google Calendar API, Microsoft Graph (Outlook), iCal feeds | Covers >95% of user calendars |
| **State management** | TanStack Query for server state; Zustand for ephemeral UI state | Minimal; no global store religion |
| **Styling / UI** | Tailwind CSS + shadcn/ui base; custom calendar/grid components | Fast to iterate, no design-system lock-in |
| **Validation** | Zod | Shared input schemas across client, server actions, and AI tool definitions |
| **Package manager** | pnpm | Fast, disk-efficient, easy to grow into a monorepo later |
| **Tests** | Vitest (unit) + Playwright (E2E) | Standard for the TS ecosystem |
| **Lint / format** | ESLint + Prettier | Standard |
| **CI** | GitHub Actions | Standard |
| **Observability** | OpenTelemetry → Datadog (or Honeycomb) for traces; Sentry for errors | Engine placement decisions need traces to debug |
| **Analytics / flags** | PostHog | Tracks PRD success metrics + feature flags in one tool |
| **Local dev infra** | `docker-compose` with Postgres on **port 5433** (non-default) | Avoids conflicts with any Postgres already on 5432 on the contributor's machine |
| **Hosting (qa)** | Vercel (Next.js) + Neon (Postgres) | Fast deploys, free preview environments per branch |
| **Hosting (prod)** | **TBD** — defined when we approach launch | Likely Vercel + Neon, but evaluate Fly.io / RDS based on cost and data residency at the time |

---

## 3. Decision Rationale

### 3.1 Why TypeScript end-to-end

Tempo's hardest cross-cutting concern is that the **same scheduling logic must run on the server and in the browser** — the client needs to replan instantly on user edits, while the server needs to replan on calendar webhooks and scheduled jobs. A single language with a shared engine module is the only way to get this without duplicating code or paying a server round-trip on every edit.

TypeScript also gets us a mature UI ecosystem (calendars, drag-and-drop, virtualization) and first-class LLM SDKs.

**Trade-off accepted:** if the engine ever needs heavy combinatorial search, we'll port the hot path to Rust + WASM. The engine module is designed as pure functions specifically to make that swap localized.

### 3.2 Why PostgreSQL

The data model is clearly relational (plans → anchors → tasks → dependencies → recurrences → completions) and benefits from:
- Foreign keys and check constraints to enforce **layer precedence** at the storage layer.
- JSONB for the parts that are genuinely flexible (energy profile config, AI metadata).
- Partial indexes for queries like "all open fluid tasks for plan X within window Y."

Document and graph databases were considered and rejected — Tempo's hardest queries are exactly what relational stores are good at.

### 3.3 Why Drizzle ORM

Drizzle gives us type-safe SQL without hiding it. The engine code reads from "a SQL database" and we never need to fight an ORM's abstraction when a query gets non-trivial. Migrations are first-party (Drizzle Kit), schema lives in TypeScript, and the runtime overhead is minimal.

Prisma was the alternative — more polished DX, but heavier runtime, generated client, and a worse story for raw SQL when we need it for engine queries.

### 3.4 Why Anthropic Claude as the default LLM

The AI features in §4.6 of the PRD lean on long-context reasoning (read a whole plan plus the user's intent), structured output (typed proposals for Anchored / Recurring / Fluid items), tool use (the AI calls into the engine to test placements), and explainable narrative (Explain-the-Plan).

We use:
- **Sonnet** for routine authoring, natural-language edits, progress chat.
- **Opus** for one-shot heavy lifting: full template expansion, goal decomposition, postmortems.

The `AIProvider` interface ensures we can route to other providers if cost, latency, or quality changes.

### 3.5 Why Clerk for auth

Auth is undifferentiated work. Clerk handles OAuth (Google + Microsoft, which we already need for calendar sync), sessions, MFA, email verification, and has org primitives we'll want for future shared-anchor features.

The asterisk: Clerk's pricing scales with MAU. We hide it behind an internal `AuthService` interface so a future migration (Ory, Lucia, self-hosted) is bounded.

---

## 4. Environments: dev, qa, prod

Tempo runs in three target environments. Each has its own configuration shape; secrets are never committed.

### 4.1 Naming

We use **two** environment variables together:

- `NODE_ENV`: standard Node convention — `development`, `test`, or `production`. Drives framework behavior (Next.js dev server, optimized builds, test runners).
- `APP_ENV`: Tempo-specific deployment target — `dev`, `qa`, or `prod`. Drives our own config selection (which database to talk to, which Anthropic key to use, which feature flags are on).

This separation matters because **qa runs `NODE_ENV=production` builds against a non-prod backend** — they're different concepts.

| `APP_ENV` | Typical `NODE_ENV` | Postgres | Clerk instance | Anthropic key | Inngest |
|-----------|--------------------|----------|----------------|---------------|---------|
| `dev` | `development` | Local docker container on **5433** | Clerk `development` instance | Dev key (low rate limit) | Inngest dev server (local) |
| `qa` | `production` | Neon QA branch | Clerk `development` instance (separate app) | Shared QA key | Inngest cloud — `qa` environment |
| `prod` | `production` | **TBD** | Clerk `production` instance | Prod key | Inngest cloud — `prod` environment |

### 4.2 Config Files

Committed (no secrets, shape only):

```
.env.development          # defaults for local dev
.env.qa                   # defaults for QA deploys
.env.production           # defaults for prod deploys (TBD values fenced)
.env.example              # the canonical list of every variable, with comments
```

Gitignored (secrets):

```
.env.local                # contributor-local overrides for dev
.env.qa.local             # not used in CI; only for local QA debugging
```

### 4.3 Loading Order

We follow Next.js conventions, with one addition: a `loadEnv(APP_ENV)` helper at server startup that reads `.env.${APP_ENV}` first, then layers in `.env.local` only when `APP_ENV=dev`. QA and prod **never** load `.env.local` files — their secrets come from the deploy platform (Vercel project env, secret manager).

### 4.4 Validation

Every variable that the app reads is declared in a single Zod schema (`src/env.ts`). The app **fails fast at boot** if any required variable is missing or malformed. This is enforced equally in dev, qa, and prod — there are no "optional in dev only" footguns.

### 4.5 What's Different Between Environments

Per environment, these things vary independently:

- **Database connection** — host, port, credentials, SSL mode.
- **Clerk publishable + secret keys** — tied to the Clerk instance.
- **Anthropic API key** + default model (we may run `claude-haiku` in dev for speed/cost).
- **Inngest signing key + event key.**
- **External calendar OAuth client IDs/secrets** — Google/Microsoft apps differ between non-prod and prod.
- **Feature flag defaults** — destructive features (replan auto-apply, AI auto-edit) default to *off* in dev/qa to avoid surprises during testing.
- **Log level** — `debug` in dev, `info` in qa, `warn` in prod.
- **Telemetry destination** — qa and prod both ship to Datadog, with environment tagging; dev is local-only by default.

---

## 5. Local Development

### 5.1 docker-compose

A single `docker-compose.yml` at the repo root brings up Postgres and (later) any other service the webapp needs locally.

Key choices:

- **Postgres image:** `postgres:16-alpine`.
- **Host port:** `5433` (mapped to container port `5432`). Chosen because **5432 is frequently already in use** — it's the default for Postgres installs, Supabase CLI, Homebrew services, etc. Using 5433 means a contributor with another Postgres running locally can clone Tempo and `docker-compose up` without port conflicts.
- **Volume:** named volume for data (`tempo_pgdata`) so resetting the container does not nuke local data unless explicitly requested (`docker-compose down -v`).
- **Healthcheck:** `pg_isready` so dependent services wait for Postgres to actually accept connections.
- **Default credentials:** `tempo` / `tempo` / database `tempo_dev` — these are dev-only and the doc states it explicitly.

The connection string used by the Next.js app in dev is therefore:

```
postgres://tempo:tempo@localhost:5433/tempo_dev
```

### 5.2 Migrations

`drizzle-kit` runs migrations against the local container. The dev workflow is:

1. `docker-compose up -d postgres`
2. `pnpm db:migrate` (alias for `drizzle-kit migrate`)
3. `pnpm dev`

QA migrations run automatically on deploy via a CI step. Prod migration policy will be defined alongside prod hosting.

### 5.3 Seeding

A `pnpm db:seed` script populates the local DB with a small but realistic dataset (one user, two plans, mix of anchored/recurring/fluid tasks). The seed script is idempotent and dev-only.

### 5.4 What's *not* in docker-compose

- Clerk — uses the hosted dev instance.
- Anthropic — calls real Claude with a dev key.
- Inngest — uses the local Inngest dev server (a binary, not a container, today).
- Calendar APIs — calls real Google/Microsoft sandbox apps.

We deliberately avoid mocking these locally. Mocked external services drift from reality and hide bugs. The dev experience pays a small connectivity cost in exchange for fewer "works on my machine" surprises.

---

## 6. Repo Structure (webapp v1)

```
tempo/
├── docker-compose.yml
├── .env.example
├── .env.development
├── .env.qa
├── .env.production         # values TBD; shape committed
├── package.json
├── pnpm-workspace.yaml     # in place from day 1, even with one app
├── apps/
│   └── web/                # the Next.js app
│       ├── src/
│       │   ├── app/        # App Router routes
│       │   ├── components/
│       │   ├── server/     # server actions, tRPC routers
│       │   ├── lib/        # shared utilities
│       │   ├── engine/     # the constraint engine (will move to packages/ later)
│       │   ├── ai/         # AIProvider interface + Anthropic implementation
│       │   ├── db/         # Drizzle schema + migrations
│       │   └── env.ts      # Zod-validated env loader
│       ├── tests/
│       ├── drizzle.config.ts
│       └── package.json
└── README.md
```

`pnpm-workspace.yaml` is in place from day 1 — it costs nothing now and means we don't have to reorganize when `packages/engine` and `packages/domain` get extracted.

---

## 7. Risks & Things We Will Revisit

| Risk | Trigger to revisit | Likely action |
|------|--------------------|---------------|
| Engine performance in TypeScript | p95 replan time >500ms with realistic plans | Port hot path to Rust + WASM behind the existing engine interface |
| Clerk pricing | MAU growth pushes Clerk above ~$1k/mo | Move to a self-hosted alternative (Ory, Lucia, or similar) |
| Single LLM dependency | Anthropic outage, pricing shift, or quality regression | Activate a second `AIProvider` implementation |
| Local Inngest binary friction | Contributors complain or the binary lags Inngest cloud | Containerize Inngest dev server in docker-compose |
| Calendar API rate limits | Google/Microsoft quota throttling on heavy sync users | Move calendar polling to a dedicated Inngest function tier with backoff and per-user quotas |
| Prod hosting choice deferred too long | First QA stability data lands and prod target is still undefined | Lock prod hosting decision before the first private beta |

---

## 8. Out of Scope for This Document

- Specific Drizzle schema / table definitions.
- Engine algorithm specification (heuristic order, pruning rules, energy-profile math).
- Prompt templates for the AI layer.
- Specific UI component inventory.
- Pricing, billing, and subscription stack.
- Mobile and desktop surfaces.
- Compliance posture (SOC2, GDPR data flows).

Each of these gets its own document once webapp v1 scope is locked.

---

## 9. Open Questions

These are decisions deferred deliberately — picking now would be premature:

1. **Where prod runs.** Vercel + Neon is the obvious choice given the qa setup, but data residency, cost at scale, and any enterprise asks may push us to Fly.io + RDS or a hybrid.
2. **One database per user vs. shared multi-tenant Postgres.** Per-user (Turso/SQLite-per-tenant style) simplifies sync and isolation; shared is operationally cheaper. Pick based on early scale numbers.
3. **Where the engine runs by default for AI-driven edits** — client-side (faster, but the AI must round-trip through the client) or server-side (simpler control flow). Likely server for AI changes, client for direct user edits.
4. **When to extract `packages/engine` and `packages/domain`** out of `apps/web`. Cheap to do early, cheap to defer until a second surface (mobile) starts needing them.
