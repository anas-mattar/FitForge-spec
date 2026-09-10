# Roadmap — FitForge

> **This file carries no implementation authority.** Agents implement only from an
> approved `specs/NNN-name/spec.md`; this roadmap only says what to spec next. A roadmap
> row is never a requirement — if an agent is asked to implement from this file, it must
> stop and ask for a spec (constitution I).

## Inventory *(generated — regenerate freely)*

Every screen in the visual reference and every capability named in the product logic.
Mechanical, no judgment.

**Generated from**: `docs/design/fitforge-prototype.html` (the `data-screen` sections) and
`docs/product/fitforge-logic.md`
**Generated on**: 2026-09-09 — **by**: manual enumeration during adoption

| Inv # | Screen / capability | Source |
|---|---|---|
| INV-001 | Sign in / register | `docs/design/fitforge-prototype.html` (`signin`) |
| INV-002 | Today — planned workout, streak, weekly volume | `docs/design/fitforge-prototype.html` (`today`) |
| INV-003 | Exercise library — grid, filter by muscle group / gear / difficulty | `docs/design/fitforge-prototype.html` (`library`) |
| INV-004 | Exercise detail — instructions, gear, muscles, your history, PR | `docs/design/fitforge-prototype.html` (`exercise`) |
| INV-005 | Program list | `docs/design/fitforge-prototype.html` (`programs`) |
| INV-006 | Program day detail — planned exercises | `docs/design/fitforge-prototype.html` (`program`) |
| INV-007 | Program builder | `docs/design/fitforge-prototype.html` (`builder`) |
| INV-008 | Active session — set rows, rest timer, finish | `docs/design/fitforge-prototype.html` (`session`) |
| INV-009 | History — session list and immutable session detail | `docs/design/fitforge-prototype.html` (`history`) |
| INV-010 | Progress — volume chart, PR table, body metrics | `docs/design/fitforge-prototype.html` (`progress`) |
| INV-011 | Profile and unit preference | `docs/design/fitforge-prototype.html` (`profile`) |
| INV-012 | Exercise / gear / muscle-group catalog administration | `docs/product/fitforge-logic.md` §2 |
| INV-013 | Session adjustment (additive correction of a completed session) | `modules/training/training-invariants.md` §1 |
| INV-014 | Derived training arithmetic — volume, Epley 1RM, streak, adherence | `docs/product/fitforge-logic.md` §4 |

## Roadmap *(authored — humans only, never regenerated)*

Thin vertical slices: foundations first, reads before writes. Each row is one feature owned
end to end by one developer across both repositories — and reviewed by the other. Ownership
alternates so neither developer becomes the only person who understands a tier.

Status flow: `idea → specified → in progress → shipped → dropped`

| Feature | Covers (Inv #) | Priority | Status | Owner | Spec |
|---|---|---|---|---|---|
| Solution scaffold (both repositories, architecture ADR in `plan.md`, proven gates) | — | P1 | shipped | anas.m | `specs/001-solution-scaffold/` |
| Identity and member profile (register, sign in, session held by the BFF) | INV-001, INV-011 | P1 | in progress | anas.m | `[specs/002-identity-member-profile/]` |
| Exercise library, read-only (seeded catalog, list, filter, detail) | INV-003, INV-004 | P1 | idea | ahmad | — |
| Program read (assigned program, day detail, today's planned workout) | INV-002, INV-005, INV-006 | P1 | idea | anas.m | — |
| Exercise catalog administration (first write slice, first migration beyond scaffold) | INV-012 | P2 | idea | ahmad | — |
| Program authoring (builder) | INV-007 | P2 | idea | anas.m | — |
| Session logging (start, log sets, rest timer, finish) | INV-008 | P1 | idea | ahmad | — |
| Session history (list plus immutable detail) | INV-009, INV-013 | P2 | idea | ahmad | — |
| Progress dashboard (volume, personal records, streak, adherence) | INV-010, INV-014 | P2 | idea | anas.m | — |
| Body-metrics log | INV-010 | P3 | idea | ahmad | — |
| Unit preference (kg / lb display) | INV-011 | P3 | idea | anas.m | — |

## Decisions log *(authored)*

- 2026-09-09 Scope fixed to workout and exercise only. No store, orders or payments; no
  nutrition tracking; no social feed. "Outfit" in the original brief is modelled as `Gear`
  attached to an exercise (what you need to perform it), not as merchandise.
- 2026-09-09 Reads before writes applied per domain area: the exercise library and program
  read slices ship before catalog administration and program authoring; session logging is
  the exception the sequence cannot avoid — it is the product, and nothing about it is
  read-only. It is therefore sequenced after four smaller features have taught the team the
  ritual, not first.
- 2026-09-09 Session logging and session history are separate features although they share a
  screen family: history reads the immutable record and introduces `SessionAdjustment`
  (invariant §1), which is a different risk class from writing live sets.
- 2026-09-09 Ownership alternates by feature, and each developer reviews the other's. Neither
  developer owns a tier: FitForge has two people, and a tier owner is a single point of
  failure with no bus factor.
- 2026-09-09 The last two rows are candidates for the Micro lane (constitution X): both are
  small, bounded, and touch no schema beyond one column. The lane is declared in the
  mini-spec when the feature is claimed, not here — a roadmap row carries no authority.
- 2026-09-10 Feature 002 claimed and declared **Critical**, not the Standard this row
  originally implied. `docs/sdlc/critical-delivery.md` ("When a feature MUST be Critical")
  names authentication and authorization outright, and 002 is both; it also carries the
  first destructive operation in the product (account deletion, invariant 10) and the
  member scoping that invariant 2 calls a defect of the highest severity. The level is
  chosen per feature at creation and declared in `spec.md`, and a roadmap row carries no
  authority — so this is a correction of an expectation, not an amendment. What it costs:
  a rollback plan before phase 1, an item-by-item invariant pass in both reviews, retained
  audit evidence, and **every phase gate run live by a human** — no `ci-held`, no batching,
  which is how 001 was certified and will not be available here.
- 2026-09-10 002's scope excludes the profile screen's gear section, though the screen is
  one of its two visual references. Gear is the exercise library's catalog (INV-003,
  INV-012, owner ahmad); building it here would claim territory 003 needs and duplicate a
  catalog neither feature would then own. 002 delivers the preferences and account halves
  of the screen; the gear section arrives with 003 and the deviation is recorded in the
  Visual Compliance Loop rather than silently skipped.
