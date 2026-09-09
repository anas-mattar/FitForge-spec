# FitForge Training Domain Invariants

> **Constitutional force**: principle V of `.specify/memory/constitution.md` points here.
> Every rule below binds every feature, every phase, and every tier. A plan that violates one
> stops and is reported; it is never quietly justified in code review.
>
> The reasoning behind these rules, and everything that is *not* a rule, lives in
> **docs/product/fitforge-logic.md** — that document changes through a normal feature; this
> one changes only by constitutional amendment.

## 1. Completed training is immutable

A `WorkoutSession` in status `completed` or `abandoned`, and every `SetEntry` belonging to it,
MUST NOT be updated or physically deleted. Corrections MUST be additive — a
`SessionAdjustment` row that read paths apply over the original — and are themselves
auditable.

**Rationale**: a training log that can be rewritten is not a log. Progress claims, personal
records and every chart in the product are only worth what the underlying history is worth,
and members do go back and "fix" a bad day.

## 2. Every record belongs to exactly one member, and never leaves them

Every `WorkoutSession`, `SetEntry`, `BodyMetric`, `SessionAdjustment`, custom `Exercise` and
member-owned `Program` MUST carry a member reference. Every read path MUST filter by the
authenticated member; a query that can return another member's training data is a defect of
the highest severity, not a bug to schedule.

**Rationale**: body weight, measurements and training frequency are personal data. There is no
sharing feature in this product, so there is no legitimate cross-member read at all — which
makes the rule absolute and cheap to enforce.

## 3. Master data is soft-deleted

`Exercise`, `Gear`, `MuscleGroup`, `Program` and `ProgramDay` MUST use soft delete
(`IsDeleted` + audit fields). Physical deletion is prohibited without explicit approval in an
approved `plan.md`.

**Rationale**: historical set entries reference exercises and program days forever. Physical
deletion breaks a member's own history — the one thing they came back for.

## 4. Canonical units and time, converted only at the edge

Loads and body weights MUST be stored in kilograms, lengths in centimetres, and all instants
in UTC. Conversion to a member's display units or time zone happens in the presentation layer
only. A stored value MUST NOT be re-derived from a rounded displayed value.

**Rationale**: round-tripping through pounds is how 100.0 kg becomes 99.8 kg after three edits.
Mixed-unit storage also makes every aggregate silently wrong.

## 5. Derived numbers are never stored as truth

Session volume, estimated 1RM, personal records, weekly volume, streaks and adherence MUST be
computed from `SetEntry` rows (plus adjustments). They MAY be cached or materialized for
performance, but the cache MUST be reproducible from the set entries and MUST NOT be the
system's answer when the two disagree.

**Rationale**: a stored PR that drifts from the sets behind it is unexplainable to the member
and unfixable without a migration. The formulas live in one place
(**docs/product/fitforge-logic.md** §4), used by every consumer.

## 6. A set is physically possible

`Reps` MUST be a non-negative integer, `LoadKg` a non-negative decimal, and `Rpe`, when
present, between 6.0 and 10.0. A working (non-warm-up) set MUST have `Reps ≥ 1`. A `SetEntry`
MUST belong to exactly one session, and its `SetIndex` MUST be unique within its session and
exercise.

**Rationale**: nonsense sets poison every aggregate in §5 and are cheap to reject at the
boundary; the alternative is data cleaning forever.

## 7. Domain rules live in the API, never in the BFF or the browser

The C# API is the sole writer of training data and the sole home of these invariants. The
Next.js BFF and the browser MUST NOT enforce, duplicate, or work around any rule in this file,
and MUST NOT open a database connection. Client-side validation is a courtesy to the member,
never the enforcement.

**Rationale**: two enforcement points means two answers. This product has a real second server
tier, which is exactly where domain logic starts leaking.

## 8. Auditability and identity

Every persisted entity MUST carry the project's audit fields (created/modified, by whom) and
the project PK standard (internal `Id` + externally-exposed `PublicId`). Internal keys MUST NOT
appear in URLs, API payloads, or logs.

**Rationale**: the audit trail is what makes an additive correction (§1) meaningful, and
exposing sequential internal keys hands out an enumeration oracle over members' data.

## 9. One authoritative state per member

A member MUST NOT have two `active` `WorkoutSession` rows, nor two `active` `Program` rows, at
the same time. The transition that would create a second one MUST close the first explicitly.

**Rationale**: "which workout am I in?" has to have one answer — for the member, and for every
aggregate that attributes a set to a program day.

## 10. Personal data is minimal and deletable

FitForge MUST NOT collect health data beyond what §1–§9 name (no medical history, no
diagnoses, no third-party health-platform data without a spec that says so). Account deletion
MUST soft-delete the member and their training data and MUST make it unrecoverable after the
stated retention window.

**Rationale**: the smallest data set that makes the product work is also the smallest breach.
