# FitForge — Product Logic

The product brain: what the system is made of, how each thing behaves, and the exact
arithmetic behind every number a member sees. This document carries **no implementation
authority** — features are implemented only from an approved `specs/NNN-name/spec.md`
(constitution I). It is the shared reference those specs are written from, and the place a
disagreement about behavior is settled before it becomes a disagreement about code.

The subset of these rules that must survive any refactor lives in
**modules/training/training-invariants.md** and carries constitutional force (principle V).
Everything here that is not in that pack is ordinary product behavior: changeable by a normal
spec, in a normal feature.

**Scope**: training. Members follow programs, log what they lift, and watch it improve. There
is no store, no payments, no social feed.

---

## 1. Entities

### Member and profile

| Field | Notes |
|---|---|
| `Id` / `PublicId` | internal key + external identifier (constitution: PK standard) |
| `Email`, `DisplayName` | email is the login identity, unique, case-insensitive |
| `UnitPreference` | `metric` (kg/cm) or `imperial` (lb/in) — a **display** setting only |
| `TimeZone` | IANA name; every "day" and "week" boundary is computed in it |
| `ExperienceLevel` | `beginner` \| `intermediate` \| `advanced` — filters and defaults only |
| `Goal` | `strength` \| `hypertrophy` \| `endurance` \| `general` |

A **Profile** holds the slow-changing descriptive data (birth year, sex if given, height).
Body weight is not a profile field — it is a time series (`BodyMetric`), because members
watch it move.

### Exercise library

**MuscleGroup** — a fixed reference list: chest, upper back, lats, shoulders, biceps, triceps,
forearms, core, glutes, quads, hamstrings, calves.

**Gear** (the "outfit" an exercise needs) — barbell, dumbbell, kettlebell, machine, cable,
smith machine, resistance band, bench, squat rack, pull-up bar, mat, bodyweight. A member
records the gear available at their gym; the library can then show only what they can
actually do today. Gear is reference data, not inventory: FitForge never tracks how many
dumbbells a gym owns.

**Exercise**

| Field | Notes |
|---|---|
| `Name`, `Slug` | slug unique among non-deleted exercises |
| `PrimaryMuscle` | exactly one |
| `SecondaryMuscles` | zero or more, never containing the primary |
| `RequiredGear` | zero or more; empty means bodyweight-anywhere |
| `Difficulty` | `beginner` \| `intermediate` \| `advanced` |
| `Instructions` | ordered steps |
| `Cues` | short coaching notes shown during a set |
| `MediaRef` | optional image/animation reference |
| `OwnerId` | null = catalog exercise; set = a member's custom exercise, private to them |
| audit + soft delete | `IsDeleted`, plus the standard audit fields |

### Programs (workout plans)

```text
Program ──< ProgramDay ──< PlannedExercise ──> Exercise
```

**Program**: name, goal, `WeeksPlanned`, `DaysPerWeek`, `OwnerId` (null = system template),
`Status` ∈ `draft | active | archived`.

**ProgramDay**: `Order` (1-based within the program), `Name` ("Push A"), optional notes. Days
are a *rotation*, not calendar dates: day 3 is the third training day the member does, not
Wednesday.

**PlannedExercise**: `Order`, `TargetSets`, `TargetRepsMin`/`TargetRepsMax`, `RestSeconds`,
and a target load expressed either absolutely (`TargetLoadKg`) or relatively
(`TargetPercentOf1Rm`) — never both.

### Sessions (what actually happened)

**WorkoutSession**: `MemberId`, optional `ProgramDayId` (null = freestyle), `Status` ∈
`planned | active | completed | abandoned`, `StartedAt`, `CompletedAt`, member notes.

**SetEntry**: `SessionId`, `ExerciseId`, `SetIndex` (1-based within the session's exercise),
`Reps`, `LoadKg`, optional `Rpe` (6.0–10.0 in half steps), `IsWarmup`, `CompletedAt`.

**SessionAdjustment**: an additive correction to a *completed* session — `Reason`, the
corrected values, who made it, when. The original set entries are never rewritten (invariant
1); readers apply adjustments on top.

**BodyMetric**: `MeasuredAt`, `WeightKg`, optional circumferences. One entry per member per
day at most; a second entry the same day replaces it *by adjustment*, not by edit.

**PersonalRecord**: not stored as truth — a derived projection (§4).

---

## 2. State machines

**Session**

```text
            start (from program day, or freestyle)
   ─────────────────────────► active ──── finish ────► completed ──► (adjustments only)
                                 │
                                 ├── abandon (member) ──► abandoned
                                 └── 24h with no new set ──► abandoned (background job)
```

- `planned` exists only for sessions the member schedules ahead; a session started directly
  goes straight to `active`.
- `completed` and `abandoned` are terminal. Nothing deletes a session — an abandoned session
  with three logged sets is real training data.
- A member has **at most one `active` session at a time**. Starting a second one abandons the
  first, and the UI says so before it happens.

**Program**

```text
draft ──publish──► active ──archive──► archived
  ▲                   │
  └──── unpublish ◄───┘   (only while no session references its days)
```

- A member has **at most one `active` program**. Activating another archives the current one.
- Archiving never touches sessions already logged against its days.

---

## 3. Flows

### 3.1 Start a session

1. The member opens Today. If an active program exists, FitForge proposes the **next day in
   the rotation**: the day after the one used by their most recent completed session, wrapping
   at the end. No program → freestyle.
2. Starting creates the session `active` with `StartedAt = now (UTC)` and materializes one
   planned row per `PlannedExercise` — as *targets shown*, not as set entries. Nothing is
   logged until the member logs it.

### 3.2 Log a set

1. Member enters reps and load (in their display unit; converted to kg at the edge — invariant
   4), optionally RPE, and marks warm-up or working.
2. The set is persisted immediately with `CompletedAt`. There is no "save workout" step to
   lose.
3. The rest timer starts client-side from the planned `RestSeconds`. Timers are never
   persisted: a rest period is not data.
4. A set may be **deleted while the session is active** (it was a mis-tap). Once the session is
   `completed`, sets are immutable and corrections are adjustments.

### 3.3 Finish a session

`completed`, `CompletedAt = now`, then: session totals are computed (§4), personal records are
re-projected for every exercise the session touched, and the streak is re-evaluated.

### 3.4 Correct a completed session

The member states what was wrong (`Reason`) and supplies the corrected value. FitForge writes
a `SessionAdjustment`; every read path applies adjustments over the original entries. History
shows the corrected number with a marker, and the original stays retrievable.

### 3.5 Progress

Three questions, answered from set entries alone: *am I lifting more?* (estimated 1RM trend
per exercise), *am I doing enough?* (weekly volume per muscle group), *am I showing up?*
(streak and adherence).

---

## 4. The arithmetic

Every number below is **derived on read** from set entries; none of it is a source of truth.

| Quantity | Definition |
|---|---|
| **Set volume** | `Reps × LoadKg`. Warm-up sets are excluded from every volume figure. |
| **Session volume** | Σ set volume over the session's working sets. |
| **Estimated 1RM** | Epley: `LoadKg × (1 + Reps / 30)`. Computed only for working sets with `1 ≤ Reps ≤ 12`; above 12 reps the estimate is unreliable and FitForge shows none rather than a bad one. A single-rep set gives exactly the load. |
| **Exercise PR** | For a member+exercise: the highest estimated 1RM, the heaviest load at any rep count, and the highest single-set volume — three PRs, each with the session and date that set it. |
| **Weekly volume per muscle group** | For each working set: 100% of its volume to the exercise's primary muscle, 50% to each secondary. Summed over the member's ISO week in their own time zone. The 50% weighting is a presentation convention, stated on the chart, not a physiological claim. |
| **Streak** | Consecutive ISO weeks, ending with the current one, containing at least one `completed` session. The current week never breaks a streak until it ends. |
| **Adherence** | `completed sessions ÷ program days scheduled` for the ISO week, capped at 100%. Freestyle sessions count toward the numerator only when the program has no day left unused that week. |
| **Unit display** | Stored kg → displayed lb as `kg × 2.20462`, rounded to 0.5 lb for loads and 0.1 lb for body weight. Input converts back at the edge; the stored value is never re-derived from a rounded display value (invariant 4). |

**Rounding rule**: compute in full precision, round once at display. A total is never the sum
of rounded parts.

---

## 5. Rules that are not invariants

These are ordinary product decisions — a later spec may change any of them.

- One active program and one active session per member (§2).
- A session may be started for any day of the active program, not only the proposed one.
- Custom exercises are visible only to their owner and cannot be added to system templates.
- A member may log a set for an exercise that is not in the day's plan; it is recorded against
  the session as unplanned and counts in every total.
- Sessions with no logged sets are `abandoned`, not `completed`, whatever the member taps.
- Body weight is optional; nothing in the product requires it.
- Deleting an account soft-deletes the member and their data, and is reversible for 30 days.

## 6. Boundaries between the tiers

- The **C# API** owns the domain and the database, and is the only writer of anything in this
  document.
- The **Next.js BFF** (route handlers) holds the session cookie, calls the API, aggregates
  responses for a screen, and caches nothing that a member could see stale. It contains **no
  rule from this document** — every invariant, every formula, every state transition is the
  API's (invariant 8).
- The **web UI** renders. Unit conversion for display happens here; the value sent to the API
  is always canonical (invariant 4).

## 7. Deliberately out of scope for v1

Nutrition and calories · coach/trainer accounts and client management · social feeds, sharing,
leaderboards · wearable and health-platform sync · offline logging with conflict resolution ·
video upload · gym check-in/attendance · anything involving payment.

Each of these would change the data model, so none of them is "just a screen later" — they are
future features with their own specs, and the roadmap says when.
