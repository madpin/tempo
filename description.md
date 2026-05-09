
# Product Specification: **Tempo**

### *The Constraint-Based Temporal Engine*

**Tempo** is an intelligent scheduling system that treats time as a finite resource. Unlike a traditional calendar, Tempo doesn't just store dates—it **solves** your schedule by balancing rigid life constraints, recurring obligations, and fluid project goals.

---

## 1. The Onboarding: Environment Mapping

The system begins by defining the "walls" of your schedule via an interactive wizard:

* **Variable Availability:** Input your work roster (e.g., 4-on/4-off), fixed routines (e.g., "Mondays are for gym, no work"), and seasonal changes.
  * It should have a easy input, calendar like, and also manual input
* **The Anchor Event:** Define the final deadline (e.g., The Exam, The Flight, The Launch).
* **Buffer Preferences:** Set "Energy Profiles"—tell the system if you want a 3-day "Cool-down" (no tasks) before the deadline or a "Sprint" (high intensity).

---

## 2. The Task Hierarchy (Solid vs. Fluid)

Tempo organizes your life into three distinct layers of "weight," ensuring that important appointments are never overwritten by general tasks.

### Layer A: Anchored Events (The "Boulders")

These are non-negotiable, fixed-time entries that dictate the flow of everything else.

* **Hard Appointments:** Specific dates/times (e.g., "Doctor visit on Tuesday at 2 PM").
* **Hard Deadlines:** "Must be done by" constraints (e.g., "Submit application *no later than* Friday").
* **Impact:** The system "locks" these slots first. No other tasks will be scheduled during these times.

### Layer B: Recurring Cadence (The "Pulse")

Mandatory, repetitive tasks that coexist with your goal.

* **Examples:** Weekly university labs, bi-weekly status reports, or every-7-day reviews.
* **Logic:** These repeat automatically but are "Pause-Aware" (can be toggled off for holidays or specific weeks).

### Layer C: Fluid Content (The "Water")

This is the heart of the system—tasks that need to happen, but the *when* is flexible.

* **Sequential Splitting:** "Read 30 chapters." Tempo looks at the gaps left by your **Anchored Events** and **Recurring Cadence**, then splits the chapters evenly into your remaining free time.
* **Atomic Checklists:** A "Bucket" of items (e.g., "Pack shoes," "Buy gift"). Tempo sprinkles these into the gaps of your schedule based on your available bandwidth.

---

## 3. The "Liquid" Distribution Engine

The system's primary intelligence lies in its ability to **play around** your life:

* **Dynamic Load Balancing:** If you add an "Anchored Event" (a new appointment) mid-week, Tempo instantly shifts your "Fluid Content" (chapters/packing) to other open slots.
* **Intelligent Grouping:** If you have a large block of free time between two appointments, Tempo will group more atomic tasks there to maximize your "flow state."
* **Recap & Review Triggers:** The system detects when you’ve completed a significant "chunk" of fluid tasks and automatically suggests a "Recap Session" or a "Rest Day."

---

## 4. AI-Driven Architecture (LLM Integration)

To minimize "Admin Fatigue," Tempo uses an LLM layer to build your lists for you:

* **Contextual Templates:** Type "Moving House," and the AI generates a mix of **Anchored Events** (book truck, shut off power) and **Fluid Tasks** (pack kitchen, declutter garage).
* **Requirement Breakdown:** Type "Renew Passport," and it generates the sequence: (1) Get photos, (2) Mail original, (3) Wait for return. It then calculates the best "Anchored Date" to start to meet your final deadline.
* **Progress Chat:** You can tell the AI, "I'm feeling overwhelmed this week," and it will automatically redistribute your fluid tasks to later dates, keeping only the **Anchored Events** in place.

---

### Summary of System Logic

| Task Type | Movement | Example |
| --- | --- | --- |
| **Anchored** | **Immovable** | Appointment, Hard Deadline |
| **Recurring** | **Predictable** | Weekly Lab, Sunday Prep |
| **Fluid** | **Adaptive** | Book Chapters, Packing List |

---

### How does this look to you?

I've emphasized the "Liquid Distribution" where your chapters and checklists automatically flow into the gaps created by your appointments. Would you like to dive deeper into the **UI layout** (how the user sees the "Boulders" vs the "Water") or perhaps the **logic for the LLM prompts**?
