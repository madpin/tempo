# Product Requirements Document: Tempo

*The Constraint-Based Temporal Engine*

---

## 1. Overview

### 1.1 Vision
Tempo is an intelligent scheduling system that treats time as a finite, fluid resource. Where traditional calendars passively store events, Tempo actively **solves** the user's schedule by balancing rigid life constraints, recurring obligations, and flexible project goals. Users describe what must happen, when it must happen, and what they want to accomplish — Tempo continuously arranges everything else.

### 1.2 Problem Statement
People juggle three fundamentally different kinds of time commitments:
- **Fixed obligations** they cannot move (appointments, deadlines).
- **Recurring rhythms** that repeat but flex (weekly classes, gym days).
- **Flexible work** with a goal but no fixed time (study chapters, packing, project tasks).

Existing tools treat all three identically, forcing the user to play scheduler. The result: forgotten tasks, over-packed days, missed deadlines, and high "admin fatigue."

### 1.3 Product Promise
Tell Tempo your walls (when you can't work), your boulders (what must happen), and your water (what you want to get done). Tempo will fit the water around the boulders, automatically and continuously, until the deadline is met.

### 1.4 Success Metrics
- **Time-to-first-plan:** user receives a complete schedule within 5 minutes of onboarding.
- **Replan latency:** schedule recomputes in under 2 seconds after any change.
- **Task completion rate:** % of fluid tasks completed by their distributed slot.
- **Deadline hit rate:** % of anchor events met without overrun.
- **Admin reduction:** average daily time spent editing the schedule (target: < 2 minutes).
- **Retention:** weekly active usage through the duration of an active anchor goal.

---

## 2. Target Users

### 2.1 Primary Personas
- **The Studier** — preparing for a fixed-date exam (bar, medical boards, certification). Has a syllabus, a deadline, and irregular availability.
- **The Mover** — relocating, traveling, or planning a wedding. Has a master checklist with dependencies and a hard date.
- **The Knowledge Worker on a Project** — shipping a launch, thesis, or proposal alongside ongoing meetings and recurring duties.
- **The Shift Worker** — non-standard schedule (4-on/4-off, rotating shifts) trying to fit life around irregular availability.

### 2.2 Anti-Personas (out of initial scope)
- Teams needing shared/collaborative scheduling.
- Users wanting a passive calendar replacement with no goal.
- Enterprise resource planning use cases.

---

## 3. Core Concepts

### 3.1 The Three Task Layers

| Layer | Name | Movement | Examples |
|-------|------|----------|----------|
| A | **Anchored Events** ("Boulders") | Immovable | Appointment at 2 PM; deadline Friday |
| B | **Recurring Cadence** ("Pulse") | Predictable | Weekly lab; bi-weekly review |
| C | **Fluid Content** ("Water") | Adaptive | Read 30 chapters; pack 12 boxes |

Tempo's defining behavior: **higher layers always displace lower layers**. Adding a boulder reshapes the water around it. Water never overwrites a boulder.

### 3.2 The Anchor Event
Every Tempo plan is organized around a single **Anchor Event** — the deadline that gives the schedule meaning (the exam, the flight, the launch). All Fluid Content is distributed across the time between *now* and the Anchor Event, respecting Anchored Events and Recurring Cadence in between.

### 3.3 Energy Profile
A user-declared shape for how intensity should ramp toward the anchor. Examples: *Sprint* (heavy load near the end), *Cool-down* (taper to rest before deadline), *Steady* (uniform distribution), *Front-load* (front-heavy).

---

## 4. Feature Specifications

### 4.1 Onboarding: Environment Mapping

**Goal:** capture the user's "walls" — the structural availability inside which everything else must fit.

**Features:**
- **F-1.1 Availability Wizard.** Step-by-step intake that collects:
  - Working hours by day-of-week.
  - Roster patterns (e.g., 4-on/4-off, rotating shifts) with start-date alignment.
  - Fixed routines (e.g., "Mondays = gym, no work").
  - Sleep window and protected personal hours.
- **F-1.2 Calendar-Style Visual Input.** Drag-to-paint availability on a weekly grid, alongside text-based input for the same data.
- **F-1.3 Manual Override.** Free-form ability to mark any specific date or range as fully unavailable (vacation), partially available, or extra-available.
- **F-1.4 Seasonal Variants.** Save named availability profiles ("Term-time," "Summer," "On-call week") and switch between them on a schedule.
- **F-1.5 Anchor Event Setup.** Define the deadline: name, date/time, type (hard deadline vs. event time), and importance.
- **F-1.6 Energy Profile Selection.** Choose a buffer style (Sprint / Cool-down / Steady / Front-load) and customize how many days the buffer spans.
- **F-1.7 Quick-Start Templates.** Pre-built onboarding flows for common scenarios (Exam Prep, Moving House, Wedding, Job Search).

**Acceptance:** at the end of onboarding, the user has a populated weekly availability map, at least one anchor event, and an energy profile.

---

### 4.2 Layer A — Anchored Events ("Boulders")

**Goal:** lock immovable commitments into the schedule first.

**Features:**
- **F-2.1 Hard Appointments.** Specific date + time + duration entries. Optional location, attendees, notes.
- **F-2.2 Hard Deadlines.** "Must complete by X" entries with no fixed time, but a fixed last moment.
- **F-2.3 Travel & Buffer Padding.** Each appointment can declare pre/post buffer (commute, prep, decompression) that Tempo reserves alongside it.
- **F-2.4 Conflict Detection.** Adding an anchored event that overlaps existing anchored events surfaces a clear conflict prompt with resolution options (move, shorten, override).
- **F-2.5 Priority Tier.** Anchored events can be marked *Critical* (Tempo refuses to schedule anything around) or *Standard* (Tempo may reduce buffer if needed).
- **F-2.6 External Calendar Import.** Pull events from Google Calendar / iCal / Outlook and treat them as Anchored. Two-way sync (configurable per calendar).

---

### 4.3 Layer B — Recurring Cadence ("Pulse")

**Goal:** model the predictable rhythm of life that repeats but can flex.

**Features:**
- **F-3.1 Recurrence Rules.** Daily, weekly, bi-weekly, monthly, every-N-days, custom rule (e.g., "every other Tuesday and Thursday").
- **F-3.2 Pause-Aware Recurrence.** Each recurring item can be paused for a date range (holidays, illness, travel) without losing its rule.
- **F-3.3 Skippable vs. Strict.** Recurring items declare whether a missed instance must be made up later (Strict) or simply skipped (Skippable).
- **F-3.4 Drift Window.** Recurring items can declare a flex window (e.g., "weekly review can land any day Sat–Mon") that the engine uses to avoid clashes.
- **F-3.5 Linked Recurrences.** Group related items so they always fall on the same day (e.g., "lab + lab report" as one block).

---

### 4.4 Layer C — Fluid Content ("Water")

**Goal:** auto-distribute flexible work across the available gaps.

**Features:**
- **F-4.1 Sequential Splitting.** A volume goal ("Read 30 chapters", "Run 200 km", "Watch 12 lectures") that Tempo divides into evenly-spaced units across the available time.
- **F-4.2 Atomic Checklists.** A bucket of discrete items ("Pack shoes", "Buy gift", "Email landlord") that Tempo sprinkles into open slots based on each item's estimated bandwidth.
- **F-4.3 Per-Task Estimates.** Each fluid item carries a *variable* estimated duration (no fixed minimum granularity — a task can be 5 minutes or 5 hours) and an optional cognitive load tag (Light / Focus / Deep).
- **F-4.4 Dependencies.** Tasks can declare prerequisites ("Mail original passport" can't start before "Get photos" is done). The engine respects ordering when distributing.
- **F-4.5 Min/Max Daily Cap.** Per project, the user can set "no more than X hours/day" or "at least Y units/day" to control density.
- **F-4.6 Earliest-Start / Latest-Finish.** Per-task fences that constrain where a fluid item may land.
- **F-4.7 Effort vs. Calendar Pacing.** User chooses whether splits are by *count* (chapters) or by *time* (90-minute study blocks).

---

### 4.5 The Liquid Distribution Engine

**Goal:** continuously arrange Layer C around Layers A & B in a way that feels intelligent, not mechanical.

**Features:**
- **F-5.1 Dynamic Replan.** Any change (new appointment, completed task, pushed deadline) instantly re-flows the fluid content. The user never presses "regenerate."
- **F-5.2 Intelligent Grouping.** Large free blocks attract high-focus tasks; small free blocks attract atomic checklist items. Cognitive-load tags drive the match.
- **F-5.3 Energy-Profile Shaping.** The engine tapers, sprints, or steadies the workload according to the chosen profile.
- **F-5.4 Cool-Down Buffer.** Optional pre-anchor zone that the engine treats as locked rest time (no fluid tasks scheduled).
- **F-5.5 Soft Preferences.** Per-task preferences ("morning only," "weekends only," "not after 7 PM") that the engine honors when feasible and reports when violated.
- **F-5.6 Recap Triggers.** When a meaningful chunk of fluid work has been completed, the engine surfaces a Recap Session or Rest Day suggestion.
- **F-5.7 Slack Visualization.** The schedule visibly shows how much slack remains before the anchor — the user always knows whether they're on track, ahead, or behind.
- **F-5.8 Stretch & Compress Modes.** When the user falls behind, Tempo offers options: extend the anchor, compress remaining work into denser slots, or drop optional items.
- **F-5.9 Non-Destructive Replan.** Replanning does not lose user-marked progress, completion state, or in-flight task notes.
- **F-5.10 Placement-Pattern Surfacing.** When the user repeatedly ignores or moves fluid items out of a particular slot, Tempo surfaces the pattern back to the user ("you've moved morning study sessions 4 times — want to mark mornings as unavailable?") rather than silently learning around it. The user remains in control of availability rules.
- **F-5.11 Insufficient-Time Alerts.** When constraints make the plan infeasible — e.g., two anchors competing for the same scarce window, or fluid volume exceeding remaining capacity — Tempo halts auto-distribution for the affected items and surfaces a clear "not enough time" alert with the specific shortfall and resolution options (extend anchor, drop scope, increase availability).
- **F-5.12 Past-Deadline Handling.** When an anchor or hard deadline passes without completion, Tempo prompts the user with options (extend, archive, mark complete-with-slip, drop). The user can configure a *default action* in settings to apply automatically when they don't respond within a chosen window.

---

### 4.6 AI-Driven Authoring (LLM Layer)

**Goal:** eliminate the cost of typing in lists by letting the user describe goals in natural language.

**Features:**
- **F-6.1 Contextual Templates.** "Moving House" → AI proposes a complete plan with Anchored events (book truck, shut off power) + Fluid tasks (pack kitchen, declutter garage). User reviews before commit.
- **F-6.2 Requirement Breakdown.** "Renew Passport" → AI generates ordered steps (photos → mail → wait → receive), estimates lead times, and back-plans the anchored start date from the deadline.
- **F-6.3 Goal Decomposition.** "Pass the bar exam in 3 months" → AI proposes a study breakdown by subject with sequential splitting per topic.
- **F-6.4 Progress Chat.** Conversational interface: "I'm overwhelmed this week" → AI redistributes fluid load to later, leaves anchored events untouched, reports what changed.
- **F-6.5 Smart Suggestions.** AI proposes new tasks the user likely missed (based on template and current state) without inserting them silently.
- **F-6.6 Natural-Language Edit.** "Move my cardio sessions to mornings" or "add a 30-minute review every Friday" without touching the form UI.
- **F-6.7 Explain-the-Plan.** User can ask "why is this task on Tuesday?" and get a plain-English explanation referencing the constraints involved.
- **F-6.8 Reviewable AI Output.** Every AI-generated change is presented as a diff (added / removed / moved) before the user accepts.

---

### 4.7 User Interface

**Goal:** make the three-layer model legible at a glance.

**Features:**
- **F-7.1 Layered Calendar View.** Visual distinction between Boulders (solid blocks), Pulse (patterned blocks), and Water (light, flowing fills).
- **F-7.2 Anchor Countdown.** Persistent indicator of distance to the next anchor event.
- **F-7.3 Today View.** Linear list of today's commitments, ordered by time, with quick-complete affordance for fluid items.
- **F-7.4 Week & Month Views.** Standard calendar layouts with the layered styling preserved.
- **F-7.5 Project View.** Grouped by goal/anchor — see all fluid + recurring items contributing to one outcome, with completion progress.
- **F-7.6 Density Heatmap.** A long-range view showing how loaded each day is between now and the anchor — surfaces over- and under-packed periods.
- **F-7.7 Drag-to-Reschedule.** Direct manipulation of fluid items; the engine reflows the rest.
- **F-7.8 Quick Add.** Single-keystroke entry that infers layer from phrasing ("@2pm Tuesday" → anchored; "every Thursday" → recurring; bare verb → fluid).
- **F-7.9 Empty-State Guidance.** Each view explains the next step when content is missing (no anchor, no availability, no fluid tasks).

---

### 4.8 Notifications & Reminders

**Features:**
- **F-8.1 Pre-Event Reminders.** Configurable lead time per anchored event.
- **F-8.2 Daily Brief.** Morning summary of today's boulders, recurring items, and recommended fluid work.
- **F-8.3 End-of-Day Recap.** Evening prompt to mark progress and adjust tomorrow's load.
- **F-8.4 Drift Alerts.** Proactive warning when current pace puts the anchor at risk. **Default cadence:** on every login, Tempo surfaces a digest of everything that has happened since the last login (drift, completed work, replans, missed deadlines). User can override the cadence to *daily* or *weekly* in settings.
- **F-8.5 Replan Notifications.** Concise summary of what shifted after a major recompute, batched into the next login digest unless the user has opted into real-time replan alerts.
- **F-8.6 Quiet Hours.** No notifications during declared sleep / personal time.
- **F-8.7 Insufficient-Time Alert (cross-plan).** When two or more active anchors compete for the same scarce window, Tempo raises a high-priority alert naming the conflicting anchors and the size of the shortfall, and pauses fluid auto-distribution for the affected range until resolved.

---

### 4.9 Review, Recap & Reflection

**Features:**
- **F-9.1 Auto-Recap Sessions.** When a chunk of fluid work completes, Tempo offers a short review block.
- **F-9.2 Weekly Review.** Scheduled retrospective: what completed, what slipped, what to reshape.
- **F-9.3 Anchor Postmortem.** After an anchor passes, generate a summary of the run (predicted vs. actual, slips, energy profile fit).
- **F-9.4 Streaks & Momentum.** Lightweight reinforcement for sustained completion, opt-in.

---

### 4.10 Settings & Customization

**Features:**
- **F-10.1 Multiple Anchor Plans.** Run more than one active anchor (e.g., "exam in June" + "move in March"). When plans compete for the same scarce window, Tempo does **not** silently rebalance — it raises an insufficient-time alert (see F-5.11 / F-8.7) and lets the user choose how to resolve.
- **F-10.1.1 Default Past-Deadline Action.** Per-plan setting that controls what happens when a deadline passes without completion: *Always Prompt* (default), *Auto-Extend by N days*, *Auto-Archive*, or *Auto-Mark Complete-with-Slip*. The default action is applied only after a user-configurable grace window.
- **F-10.1.2 Login-Digest Cadence.** Override the default "on login" digest cadence to *Daily* or *Weekly*.
- **F-10.2 Per-Plan Energy Profiles.** Different profiles per anchor.
- **F-10.3 Time Zone Awareness.** Travel-resilient: schedule respects current time zone but anchor times remain fixed.
- **F-10.4 Localization.** Date/time formats, weekday start, language.
- **F-10.5 Theming & Density.** Compact / comfortable views; light / dark.
- **F-10.6 Export.** Plan exportable to standard calendar formats and as a printable PDF.
- **F-10.7 Privacy Controls.** Per-item visibility for items synced to external calendars.

---

### 4.11 Data, Sync & Reliability

**Features:**
- **F-11.1 Offline Edits.** User can review and edit the plan without connectivity; changes reconcile on reconnect.
- **F-11.2 Multi-Device Sync.** Plan stays consistent across mobile, web, and desktop.
- **F-11.3 Edit History.** Per-item change history with restore.
- **F-11.4 Undo Replan.** Any automatic redistribution can be reverted with a single action.
- **F-11.5 Backup & Export.** Full plan export for portability.

---

## 5. Cross-Cutting Behaviors

### 5.1 Layer Precedence (Invariant)
At all times: **Anchored > Recurring > Fluid.** The engine must never violate this ordering, even under conflicting user input — instead, surface the conflict and ask.

### 5.2 Transparency
Every automatic decision (why a task moved, why it landed Tuesday, why a day is full) must be inspectable in plain language.

### 5.3 Reversibility
Every AI-driven or engine-driven change must be reversible in one step.

### 5.4 Trust-Building Defaults
On first use, Tempo proposes — does not impose. Auto-apply behaviors are opt-in until the user has seen Tempo behave correctly several times.

---

## 6. Out of Scope (v1)

- Multi-user / shared schedules.
- Team capacity planning.
- Native time tracking against estimates.
- Habit tracking as a primary surface (recurring tasks are not habits).
- Native video/meeting integration (Zoom, Meet) beyond calendar links.
- Goal-setting frameworks (OKRs, KPIs).

---

## 7. Future Considerations

- **Shared anchors** — couples/teams aligning around one deadline (a wedding, a launch).
- **Adaptive learning** — Tempo learns each user's actual completion rate per task type and refines estimates.
- **Wearable signals** — sleep / stress data informs energy profile in real time.
- **Voice-first authoring** — full parity with the chat-based AI authoring through speech.
- **Public templates marketplace** — share onboarding templates across users.

---

## 8. Resolved Design Decisions

The following questions were raised during PRD drafting and have been resolved into the feature spec above. They are recorded here for traceability.

| # | Question | Decision | Where it lives |
|---|----------|----------|----------------|
| 1 | How does Tempo behave when multiple anchors compete for the same scarce window? | **Alert the user about the lack of time.** Tempo does not silently rebalance across plans; it surfaces the shortfall and lets the user resolve. | F-5.11, F-8.7, F-10.1 |
| 2 | What is the right granularity for an atomic fluid task? | **Variable.** No fixed minimum — a task can be 5 minutes or 5 hours, set per-task. | F-4.3 |
| 3 | When the user repeatedly ignores a fluid placement, should Tempo learn around it or surface the pattern? | **Surface back.** Tempo flags the pattern and lets the user decide whether to update availability rules. | F-5.10 |
| 4 | How aggressive should drift alerts be before they become noise? | **On login** by default — show a digest of everything since the last login. User can override to *daily* or *weekly*. | F-8.4, F-10.1.2 |
| 5 | For deadlines that pass without completion, what is the default behavior? | **Prompt by default**, but the user can configure a default action (auto-extend / auto-archive / auto-mark complete-with-slip) per plan. | F-5.12, F-10.1.1 |
