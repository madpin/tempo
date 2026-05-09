# Tempo Webapp — Implementation Plan

*Companion to `PRD.md` and `TECHNICAL_DECISIONS.md`. Describes the build order: which features ship in which milestone, what backend and frontend pieces each milestone needs, and what we deliberately defer. This is a sequencing plan, not a schedule — no week numbers, no story points.*

---

## 0. How to Read This

The plan is organized into **six milestones**, M0 through M5. Each milestone is a meaningful internal release: it can be demoed end-to-end, it teaches us something about the product, and the next milestone depends on it.

Within each milestone we split work into:

- **Backend** — Drizzle schema, server actions / tRPC routers, Inngest jobs, external integrations, the engine.
- **Frontend** — App Router routes, UI components, client interactions.
- **Cross-cutting** — auth, observability, tests, env config, anything that isn't strictly client or server.

Two principles guide the sequencing:

1. **Walk before running the engine.** The engine is the product's moat, but it's also the riskiest piece. We build manual data entry for all three task layers *before* automating distribution, so when the engine arrives it slots into a working skeleton instead of being the foundation.
2. **AI features layer onto a working engine, not the other way around.** The LLM produces tasks; the engine places them. If the engine is unreliable, AI output is worse than useless. Engine first, AI second.

---

## 1. Working Assumptions

- The team is small (1–3 engineers) building from scratch.
- We are shipping the **webapp only** in this plan. Mobile, desktop, and the cross-platform engine package come later and have their own plans.
- We use the stack in `TECHNICAL_DECISIONS.md`: Next.js 15 / TS / Postgres / Drizzle / Clerk / Anthropic / Inngest / pnpm / Tailwind + shadcn.
- Prod hosting is deferred until before M5. Dev runs locally; QA runs on Vercel + Neon.
- Every milestone ends with a working app on `dev` and a successful deploy to `qa`.

---

## 2. Milestones at a Glance

| # | Milestone | Goal | Defining capability |
|---|-----------|------|----------------------|
| **M0** | Foundations | Empty but production-shaped app, deployable, authenticated | A signed-in user lands on a blank dashboard |
| **M1** | Walking Skeleton | Manual entry for all three task layers, rendered on a calendar | A user can hand-build a plan and see it on a week view |
| **M2** | The Engine | Automatic distribution of fluid tasks; instant replan on edit | Adding an appointment instantly reflows the user's chapters |
| **M3** | AI Authoring | LLM creates and edits plans from natural language | "Renew my passport by July" produces a reviewable plan |
| **M4** | Sync & Signals | External calendar sync, notifications, recap, drift alerts | Google Calendar appears as anchored events; daily digest arrives at login |
| **M5** | Polish & Launch Readiness | Multi-anchor, density heatmap, prod hosting, observability | Private beta opens |

---

## 3. M0 — Foundations

**Goal:** a deployable, authenticated, empty Next.js app with the toolchain and environment story in place.

### Backend
- pnpm workspace with `apps/web` in place (even though it's the only package today).
- Next.js 15 App Router scaffold with strict TypeScript, ESLint, Prettier.
- `src/env.ts` — Zod-validated env loader that reads `APP_ENV` and fails fast on missing variables.
- Drizzle + Drizzle Kit installed; empty schema; migration tooling working against the local container.
- `docker-compose.yml` running Postgres 16 on **port 5433** with a named volume and healthcheck.
- Clerk integration: middleware-protected routes, sign-in / sign-up pages, server-side `auth()` helper.
- A single `/healthz` route that confirms DB connectivity and returns build metadata.
- GitHub Actions: lint + typecheck + unit tests on every PR; deploy to QA (Vercel + Neon) on merge to `main`.

### Frontend
- App shell with `(authed)` and `(public)` route groups.
- Empty dashboard at `/dashboard` that displays the signed-in user's name.
- Tailwind + shadcn/ui base components installed; one page using them as a smoke test.
- Light/dark theme toggle (boring, but it exercises the theming pipeline).

### Cross-cutting
- `.env.example` enumerates every variable; `.env.development`, `.env.qa`, `.env.production` committed with non-secret defaults.
- `.env.local` gitignored; contributor README explains the setup flow.
- Sentry installed and capturing errors in QA.
- A README that gets a fresh contributor from `git clone` to a working dev server in under 10 minutes.

### Deferred from M0
- Any Tempo domain logic — no plans, no tasks yet.
- LLM integration.
- Background jobs (Inngest is configured but unused).

### Demo criteria
A teammate clones the repo, runs `docker-compose up`, `pnpm install`, `pnpm dev`, signs in with Clerk, and lands on an empty dashboard. The same flow works against the QA deploy.

---

## 4. M1 — Walking Skeleton

**Goal:** the user can manually build a complete plan — anchor, availability, anchored events, recurring cadence, fluid tasks — and see it laid out on a calendar. **No automatic distribution yet.** Fluid tasks render as an unsorted list.

### Backend
- **Drizzle schema** for the core domain:
  - `users` (Clerk id mirror), `plans`, `anchors`, `availability_windows`, `availability_overrides`.
  - `tasks` table with a discriminator column (`anchored` | `recurring` | `fluid`) plus type-specific JSONB for fields that vary.
  - `recurrence_rules` (RRULE-shaped), `task_dependencies`, `task_completions`.
- Migration scripts and a seed script producing one realistic plan.
- Server actions / tRPC routers for full CRUD on each entity, with Zod input validation.
- Layer-precedence invariant enforced in the data layer (DB checks + service-layer guards).
- Conflict detection for anchored events (returns a structured conflict, doesn't auto-resolve).
- Energy profile stored on the plan as a typed enum + JSONB config.

### Frontend
- **Onboarding wizard** at `/onboarding`:
  - Availability step (calendar-paint UI + manual time inputs).
  - Anchor event step (date, type, importance).
  - Energy profile step (Sprint / Cool-down / Steady / Front-load).
- **Plan dashboard** at `/plan/[id]`:
  - Anchor countdown header.
  - Today view (linear list).
  - Week view (calendar grid, layered styling: solid for anchored, patterned for recurring, light fill for fluid — even though fluid placement is just round-robin for now).
- **Task creation surfaces** for each layer:
  - "New appointment" modal (Anchored).
  - "New recurring task" modal with RRULE-style picker (Recurring).
  - "New goal" modal that creates a Fluid task with a count or duration target.
- **Quick Add** input (`+` shortcut) that infers the layer from phrasing.
- **Settings page** for editing availability and energy profile after onboarding.

### Cross-cutting
- E2E test (Playwright) for: sign in → onboarding → see populated dashboard.
- Domain types extracted to a `src/domain/` directory, consumed identically by client and server.
- Calendar import is a stub button that toasts "coming in M4."

### Deferred from M1
- Automatic placement of fluid tasks (M2).
- AI-driven authoring (M3).
- Drift alerts, notifications, recap (M4).
- External calendar sync (M4).
- Density heatmap, project view, stretch & compress (M5).

### Demo criteria
A user signs up, completes onboarding, manually adds three appointments, two recurring items, and one fluid goal ("read 30 chapters by anchor date"). The week view renders all of them with correct layering. The fluid task appears as an unsorted backlog — no claim that it's been *distributed*.

### Risks
- **Schema churn.** The domain shape will keep changing through M2–M3. Build the schema with migrations from day 1 and accept that several will land before M2 ships.
- **Calendar UI scope creep.** Resist building a fully-featured calendar component. The week view in M1 only needs to *render*; drag-and-drop and rich interactions can wait until M2 when there's something worth dragging.

---

## 5. M2 — The Engine

**Goal:** the Liquid Distribution Engine v1. Fluid tasks are automatically placed in the gaps left by anchored events and recurring cadence, respecting the energy profile. Any edit triggers an instant replan.

### Backend
- **`engine/` module** in `apps/web/src/engine/` — pure functions, no I/O, no clock except via injected dependency.
  - **Inputs:** plan, availability, anchored events, recurring expansions over the planning window, fluid tasks (with estimates, dependencies, fences), energy profile.
  - **Output:** a list of `ScheduledFluidBlock`s with start/end timestamps and a placement reason for each.
- v1 algorithm: greedy heuristic with the following stages:
  1. Expand recurring rules into concrete instances inside the planning window.
  2. Subtract anchored events, recurring instances, sleep, and unavailability from the calendar to produce **gap intervals**.
  3. For each fluid task, allocate units across gaps, weighted by the energy profile shape.
  4. Honor dependencies (topological order), per-task earliest-start / latest-finish, and daily caps.
  5. Emit a placement record with a human-readable reason (used later by Explain-the-Plan).
- Replan triggers:
  - Any mutation to a plan, anchor, availability, anchored event, recurring rule, or fluid task fires a **replan job** (Inngest in QA, in-process in dev) that recomputes placements and writes them to the DB.
  - Replans are **non-destructive**: completion state, notes, and user-pinned slots are preserved.
- **Insufficient-time alert** (PRD F-5.11): when the engine cannot fit all fluid units before the anchor, it returns a structured shortfall and the API surface raises it to the UI.
- **Placement-pattern surfacing** (PRD F-5.10): every time the user moves a fluid block, log the move; an Inngest scheduled function detects 4+ moves out of the same slot per task and creates a "pattern" notification.
- **Past-deadline handling** (PRD F-5.12): a daily Inngest job inspects passed deadlines and prompts the user; per-plan default action is honored after the configured grace window.

### Frontend
- Week view now shows fluid tasks in their *placed* slots with the same layered styling.
- **Drag-to-reschedule** on fluid blocks: drop somewhere new, the engine accepts the user's pin and reflows everything else around it.
- **Replan toast** on any change ("3 chapters moved to Wed, Thu, Fri") with one-click undo (PRD F-11.4).
- **Density heatmap** — a long-range strip from today to the anchor showing daily load. Surfaces over- and under-packed days.
- **Insufficient-time banner** on the plan dashboard when the engine reports a shortfall, with the resolution options inline (extend anchor, drop scope, increase availability).

### Cross-cutting
- Engine has a thorough unit test suite. Every fixture is a JSON file (input plan + expected placements) so regressions are reviewable as diffs.
- Performance budget: p95 replan time <500ms for a plan with 200 fluid units, 50 anchored events, 5 recurring rules. Logged as a metric.
- **Decision point before M2 starts:** confirm engine algorithm strategy. v1 is heuristic; if early prototypes show poor results we evaluate a constraint solver (OR-Tools via a sidecar) before committing further.

### Deferred from M2
- Stretch & compress modes (M5).
- Multi-anchor cross-plan deconfliction (M5).
- AI-suggested reflows (M3).

### Demo criteria
A user opens a plan with 30 chapters and an anchor 30 days out. The week view shows the chapters distributed across available time. The user drags a chapter to Saturday afternoon; the rest reflow within 500ms. The user adds an unexpected appointment on Tuesday; the chapters that were on Tuesday move automatically. An undo button reverts the last replan.

### Risks
- **Heuristic quality.** The placement that is "correct" is partly a matter of taste. Plan to spend real time tuning weights with realistic test fixtures, and gather feedback from M1 dogfooders.
- **Replan latency.** TypeScript is fine for v1 sizes, but pathological inputs (many recurring rules over long horizons) can blow up. Cap planning window at a sane horizon (90 days from today by default) in v1.

---

## 6. M3 — AI Authoring

**Goal:** the user can describe goals in natural language; the LLM proposes a reviewable plan that the engine then places.

### Backend
- `AIProvider` interface in `apps/web/src/ai/` with an Anthropic implementation (Sonnet for routine work, Opus for plan generation).
- **Tool definitions** the LLM can call:
  - `propose_anchored_event`, `propose_recurring_task`, `propose_fluid_task` (return typed proposals — never written directly to the DB).
  - `query_plan` (reads current plan state for context).
  - `simulate_placement` (calls the engine in dry-run mode so the LLM can see whether a proposal fits before suggesting it).
- **Prompt assets** stored in version control as plain TS modules — not in a CMS, not loaded at runtime. Prompts are reviewable in PRs.
- **Diff API**: every AI-driven change produces a structured diff (added / removed / moved items) that the frontend renders for the user to approve, reject, or edit before commit.
- Rate limiting and per-user cost caps on the AI surface.

### Frontend
- **Template entry** ("Moving House," "Renew Passport," "Bar Exam") — a dropdown plus free-form prompt.
- **AI plan preview**: the diff view, with per-item accept / reject / edit. Nothing applies to the live plan until the user confirms.
- **Progress chat** — a side panel where the user can say "I'm overwhelmed this week" and watch the AI propose a reflow. Anchored events are visibly excluded from any chat-driven move.
- **Explain-the-Plan**: clicking any placed fluid block reveals the engine's reason in plain English ("placed Tue 10–11am because Mon and Wed are full and you marked mornings as preferred").
- **Natural-language edit bar** at the top of the plan view ("move my cardio to mornings").

### Cross-cutting
- Prompt evaluation harness: a folder of `.eval.ts` files runs canonical prompts against the model and snapshots structured outputs. Regressions show up in CI.
- Cost telemetry per AI surface (template expansion, chat turn, NL edit) shipped to PostHog so we can see which surfaces are expensive.
- Feature flag (`ai.authoring`) defaults *off* in dev/qa for users not opted in, *off* by default in prod until the AI surface is trustworthy enough.

### Deferred from M3
- Auto-applied AI edits without user review (intentionally never; the PRD requires reviewable diffs).
- Voice-driven authoring (future).
- Public templates marketplace (future).

### Demo criteria
A user types "I'm taking the bar exam in 12 weeks." The AI proposes a study breakdown by subject. The user accepts. The plan populates. Two weeks later the user says "I'm sick this week, push everything." The AI proposes a reflow. The user accepts. The user clicks one of the placed sessions and reads "placed Wednesday because Monday's lab block was full and you set Wed as a high-energy day."

### Risks
- **Prompt drift across model versions.** Treat prompts as code; pin the model version explicitly; re-run the eval harness when bumping versions.
- **AI placement disagreeing with the engine.** The flow is: AI *proposes content*, engine *places content*. Never let the AI write directly to scheduled slots — always go through the engine's simulator first.

---

## 7. M4 — Sync & Signals

**Goal:** Tempo connects to the user's real life. External calendars feed Anchored Events. Notifications, drift alerts, and recap sessions exist.

### Backend
- **Calendar adapters** (`apps/web/src/calendar/`):
  - Google Calendar (OAuth via Clerk-managed tokens; webhook subscription for change notifications).
  - Microsoft Graph (Outlook).
  - iCal subscription URLs.
- Adapter contract: ingest events as Anchored entries, tagged with a source so user-created and synced events are distinguishable.
- Two-way sync configurable per calendar (off by default; one-way ingestion is the safer M4 default).
- **Inngest jobs**:
  - Calendar polling for adapters without webhooks (iCal).
  - **Login digest** assembly: a function that builds the "since last login" summary on demand.
  - Drift detector: scheduled per plan; emits a drift event when current pace puts the anchor at risk.
  - Recap trigger: detects when a meaningful chunk of fluid work has completed and proposes a recap session.
  - Daily past-deadline sweeper.
- **Notification fanout**: web push, email (via Resend or similar). One internal `Notification` type, multiple delivery channels.
- **Quiet hours** honored per the user's availability profile.

### Frontend
- **Calendar connections** page: connect / disconnect Google, Microsoft, iCal feeds; show last-sync time and any error.
- **Login digest sheet** on first page load per session, summarizing what happened since last login (drift, completions, replans, missed deadlines, AI suggestions waiting).
- **Notification preferences** page: cadence (every login / daily / weekly), channel (push / email), quiet hours.
- **Recap card** that surfaces in the dashboard when the recap trigger fires.
- **Anchor postmortem view** for completed plans.

### Cross-cutting
- Per-user OAuth token storage with rotation and revocation.
- Backoff and quota handling on calendar APIs.
- Integration tests against Google's sandbox and Microsoft's sandbox for the OAuth + webhook flows.

### Deferred from M4
- Native mobile push (mobile app comes later).
- Slack / iMessage / SMS channels.
- Habit tracking surface (out of scope per PRD).

### Demo criteria
A user connects their Google Calendar; existing events appear as Anchored within 10 seconds. They add a meeting in Google; within a minute it appears in Tempo and fluid tasks reflow around it. They sign back in the next morning and see a digest: "Yesterday you completed 3 chapters; tomorrow looks light; you missed your weekly review on Sunday."

### Risks
- **OAuth and webhook plumbing eats more time than expected.** Build the Google adapter end-to-end *first* so the contract is real; Microsoft and iCal fall in line behind it.
- **Notification noise.** Default to login-digest only; let the user opt in to more frequent delivery, never the other way around.

---

## 8. M5 — Polish & Launch Readiness

**Goal:** secondary features the PRD calls out, plus everything required to open a private beta on a real prod environment.

### Backend
- **Multi-anchor plans**: a user can run more than one active anchor; the engine flags when two anchors compete for the same scarce window (insufficient-time alert) rather than silently rebalancing.
- **Stretch & compress modes**: when the user falls behind, present concrete options (extend anchor, compress remaining work, drop optional items) computed by the engine.
- **Linked recurrences** (PRD F-3.5).
- **Edit history & restore** (PRD F-11.3).
- **Export**: iCal export of the full plan; printable PDF.
- **Time zone handling** for travelling users — anchor times stay fixed, schedule respects current TZ.

### Frontend
- **Project view** — group all items contributing to one anchor; show progress.
- **Anchor postmortem** UI completion.
- **Streaks / momentum** widget (opt-in).
- **Localization** scaffolding (en first; structure ready for additional locales).
- **Empty-state guidance** audit across every view.
- **Accessibility audit** — WCAG 2.1 AA pass on the canonical flows.

### Cross-cutting
- **Prod hosting locked** — the deferred decision is made: Vercel + Neon, Fly.io + RDS, or hybrid. `.env.production` filled in; deploy pipeline live.
- **Observability**: OpenTelemetry traces wired through the engine's placement decisions; Datadog dashboards for replan latency, AI cost, calendar sync health.
- **Performance pass**: bundle size budget, Lighthouse scores, p95 page-load and replan latency targets met.
- **Security review**: input validation audit, Clerk session hardening, Anthropic key rotation, calendar OAuth scope minimization.
- **Backup and restore** drill on the prod database.
- **Private beta feature flags** in PostHog; invite flow.

### Deferred (post-launch)
- Shared anchors (multi-user plans).
- Adaptive learning (Tempo learns user-specific completion rates).
- Public templates marketplace.
- Mobile and desktop surfaces (separate plans).

### Demo criteria
The webapp is live on the production domain. A small group of beta users have invites. Real plans are running through the engine. Dashboards show healthy replan latency. The team can deploy a hotfix in under 30 minutes.

---

## 9. Cross-Cutting Tracks

Some work runs alongside every milestone rather than living in one of them.

### 9.1 Testing
- **Unit tests** are mandatory for the engine module from M2 onward; aim for fixture-based tests (input JSON → expected placements) that double as regression cases.
- **E2E tests** cover the canonical flow (sign in → onboarding → plan create → engine reflow) from M2 onward and grow as features land.
- **AI eval harness** lives from M3 onward; runs in CI on prompt or model changes.
- **Visual regression** for the calendar grid is opt-in until M5.

### 9.2 Observability
- Sentry from M0; OpenTelemetry tracing wired through the engine in M2; full Datadog dashboards in M5.
- Every AI call logs cost and latency; engine replans log inputs, output size, and duration.

### 9.3 Accessibility
- Baseline keyboard navigation and ARIA from M1; full WCAG 2.1 AA audit in M5.

### 9.4 Security
- Zod input validation on every server action from M0.
- Clerk session security defaults respected; no custom session handling unless absolutely necessary.
- Calendar OAuth scopes minimized to the smallest set that delivers the feature.
- Secrets never in the repo; rotation runbooks documented before M5.

### 9.5 Documentation
- README updated each milestone.
- Engine fixtures double as documentation of expected behavior.
- A short ADR for any decision that overrides `TECHNICAL_DECISIONS.md`.

---

## 10. Decision Points

Moments where the team should pause and explicitly choose:

| Before milestone | Decision |
|------------------|----------|
| M2 | Confirm engine algorithm strategy (heuristic vs. solver). Run a spike if uncertain. |
| M3 | Confirm prompt-versioning approach and model pinning policy. |
| M4 | Confirm two-way calendar sync default (we currently propose one-way for M4, two-way as opt-in). |
| M5 | Lock prod hosting. Lock observability vendor. Decide private beta size. |

---

## 11. Out of Scope for This Plan

- Mobile (React Native) build plan.
- Desktop (Tauri) build plan.
- Cross-platform engine extraction into `packages/engine`.
- Pricing, billing, and subscription stack.
- Compliance posture (SOC2, GDPR data flows).
- Marketing site.

Each gets its own document when it becomes the next priority.
