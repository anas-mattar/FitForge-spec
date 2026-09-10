# Human PR Review — 001 Solution Scaffold

**Reviewer**: Ahmad
**Date**: 2026-09-10
**AI review**: `ai-code-review-api.md`, `ai-code-review-web.md`,
`ai-code-review-governance.md` — all three **REQUEST CHANGES**, filed 2026-09-10
(`3fa2878`). Every BLOCKING finding is closed by phases 7 (`798ca64`), 8 (`7d3e257`)
and 9 (`dee58e9`), except web F6, which is escalated as a rung conflict and is not a
code fix (see `tasks.md`, "Phase 9 — rung conflict").
**Spec / plan / tasks**: `specs/001-solution-scaffold/spec.md`,
`specs/001-solution-scaffold/plan.md` (ADR-001 in §4),
`specs/001-solution-scaffold/tasks.md`, `specs/001-solution-scaffold/contracts/health.md`

> Prepared by the implementing agent so the reviewer spends their time reviewing rather
> than assembling. **Every box below is unticked and every verdict is blank on purpose** —
> the implementer does not grade its own work (constitution IX, `docs/sdlc/review-process.md`).
> Nothing here is evidence of review; it is the map of what to review.
>
> **Provenance of the approval below, 2026-09-10.** Ahmad's APPROVED was **relayed by the
> owner in conversation**. None of the four pull requests carries a GitHub review — checked
> via `gh api repos/anas-mattar/<repo>/pulls/<n>/reviews`, all four returned empty — so
> this file is the only record of it and the owner's word is its authority.
>
> The checklists below are therefore left **unticked**. Recording that Ahmad verified each
> specific item would be inventing detail nobody supplied; recording that he approved is
> reporting what the owner said. Those are different claims and this file keeps them
> apart. Anyone who needs item-level assurance should ask Ahmad to submit the review on
> the PRs, where it is attributable to him rather than to a relay.
>
> An earlier approval, relayed before the AI reviews existed, was declined here on the
> grounds that it could not have accounted for findings that had not been made yet. That
> objection no longer applies: the reviews exist, and their BLOCKING findings are closed.

## What merges

Three repositories, one feature branch each, all named `001-solution-scaffold`.

| Repo | Phase | Commit | What it delivers |
|---|---|---|---|
| fitforge-api | 1 | `47ace55` | Three projects, dependency direction, RFC 9457 handling, `/health/live`, WeatherForecast deleted |
| fitforge-api | 2 | `404a10b` | `FitForgeDbContext`, validated options, `/health/ready` with the database check and its 503 path |
| fitforge-web | 3 | `e0df410` | `globals.css` tokens, Inter, theme-before-paint, sidebar / bottom bar / header to VI-001…VI-013 |
| fitforge-web | 4 | `8d9232d` | `api-client.ts`, `/api/health`, the health indicator, mapping tests |
| FitForge-spec | 5 | `2759908` + `ae7c38d` | `docs/onboarding.md` clone-to-running, SC-005 demonstrated (`ae7c38d` is the CI-graded head) |
| fitforge-api | 6 | `cded3bd` | Readiness bounded at 2s so `degraded` stays reachable |

Governance commits ride separately and carry no `phase N` token: `5a34b5d` (spec, plan,
tasks, contract), `26d9108` (plan §5 packages), `5e4e8d8` (phase 3 VCL result), `8785678`
(contract readiness bound + phase 6), `7d3f297` (the bound is on the document), `92455d6`
(phase 7 queued), `be8daef` (phase 5 SC-005 record), `b156260` (this file), `3fa2878`
(the three AI reviews).

**Phase 7 is queued and NOT in this merge** (`tasks.md`). Decide explicitly whether the
feature merges with that gap documented, or waits.

## Where to look hardest

Superseded 2026-09-10 by the three AI reviews, which found more than this section
anticipated. Read them in this order:

1. **`ai-code-review-governance.md` F3** — every plan, tasks and contract amendment after
   the single owner approval was made by the implementing session and implemented against
   minutes later, with no recorded approver. That is a question about how this feature was
   run, and it is the owner's to answer, not the reviewer's.
2. **`ai-code-review-api.md` F1** / **governance F4** — `/health/ready` emits a
   self-contradicting document for a `Degraded` dependency, and the BFF shows "API ready"
   while a dependency is down.
3. **`ai-code-review-api.md` F2** — the gate fails on a developer machine configured the
   way `docs/onboarding.md` tells them to configure it, and CI can never show it.

The three items this section originally listed are all still open questions; the reviews
answer none of them, and two of the three now have findings attached.

## Business Review

- [ ] Behavior matches the business intent in `spec.md` (not just the letter of the FRs)
- [ ] Domain correctness verified for business-critical outputs (spot-check real figures/cases)
- [ ] Open questions / CONFIRM findings from the AI review are answered or explicitly deferred

Note: this is a scaffold. There is no training domain logic in it yet — FR-001…FR-013 are
about structure, boundaries and the health contract. `modules/training/training-invariants.md`
is guarded structurally (`FitForge.Domain` references zero packages) rather than behaviourally.

## UI Review

- [ ] Actual rendered UI compared against the visual references (not just the code)
- [ ] Loading / empty / error states behave sensibly

The Visual Compliance Loop result is recorded in `tasks.md` ("Phase 3 — Visual Compliance
Loop result"). VI-001…VI-013 all PASS; one deviation was found (bottom-bar label clearing
its box by 0.4px) and fixed. **Re-render it yourself** — the loop was measured by the agent
that wrote the code, and SC-003 asks for an empty deviation table at merge.

## Technical Review

- [ ] Code diff read end-to-end; no unrelated changes (`git diff --stat` matches the phase scope)
- [ ] Architectural compliance (constitution IV) — no unapproved patterns/packages
- [ ] Security implications considered (authz on new surface, secrets, logging)
- [ ] Migrations/schema changes are additive or their rollback is documented

Machine checks already green, and not a substitute for the above:

```text
scope-check:  PASS phase 5 commit ae7c38d (1 file(s))
scope-repos:  fitforge-api: PASS phase 6 commit cded3bd (3 file(s))
scope-repos:  fitforge-web: PASS phase 4 commit 8d9232d (8 file(s))
ritual-checks: RESULT OK (all 7 members)
```

Packages were amended in the plan before use (`26d9108`); no package entered either repo
without that amendment. There are no migrations yet — `FitForgeDbContext` has no `DbSet`.
SC-004 is verifiable by search: no database call outside `FitForge.Infrastructure`, and no
API call from browser code (`api-client.ts` is `server-only`, verified by a build probe).

## Gate Result

`**Gate Certification**: ci-held` is declared in `plan.md` §1, so each phase's gate is the
owner's recorded approval on the evidence triplet.

| Phase | Repo | Run | Conclusion | Commit sha | Owner approval |
|---|---|---|---|---|---|
| 1 | api | [34428340737](https://github.com/anas-mattar/fitforge-api/actions/runs/34428340737) | success | `47ace55` | recorded 2026-09-10 |
| 2 | api | [34429439349](https://github.com/anas-mattar/fitforge-api/actions/runs/34429439349) | success | `404a10b` | recorded 2026-09-10 |
| 3 | web | [34430480295](https://github.com/anas-mattar/fitforge-web/actions/runs/34430480295) | success | `e0df410` | recorded 2026-09-10 |
| 4 | web | [34431163998](https://github.com/anas-mattar/fitforge-web/actions/runs/34431163998) | success | `8d9232dcde1185032fcf1424b44476cdbdf727ee` | recorded 2026-09-10 |
| 5 | governance | [34433740428](https://github.com/anas-mattar/FitForge-spec/actions/runs/34433740428) | success | `ae7c38d` | recorded 2026-09-10 |
| 6 | api | [34432663594](https://github.com/anas-mattar/fitforge-api/actions/runs/34432663594) | success | `cded3bda60869c1057d8f03be4f30b0a8370de34` | recorded 2026-09-10 |
| 7 | api | [34436554259](https://github.com/anas-mattar/fitforge-api/actions/runs/34436554259) | success | `798ca647bf168d0d9023cdee134ab7dc25f0655d` | **NOT YET RECORDED** |
| 8 | api | [34436554259](https://github.com/anas-mattar/fitforge-api/actions/runs/34436554259) | success | `7d3e257` — see note | **NOT YET RECORDED** |
| 9 | web | [34437476386](https://github.com/anas-mattar/fitforge-web/actions/runs/34437476386) | success | `dee58e92ce28314c5dff1e1cddc8929b56ed11ed` | **NOT YET RECORDED** |

**Note on phase 8's evidence.** `7d3e257` has no CI run of its own: phases 8 and 7 were
pushed together, so only the branch head `798ca64` was graded. Under
`docs/sdlc/gate-command.md` that is the batch-end commit and is legitimate evidence — for
a **declared** batch. This batch was not declared; `plan.md` §1 says `Gate Batching: none`.
The runs are genuinely green and genuinely cover phase 8's code, but the paperwork says
one thing and the push did another, and that is the same shape of gap the governance
review's F2 found in phase 5's row. Certify it as a batch or re-push phase 8 alone; do not
let the row imply a run that does not exist.

Corrected 2026-09-10 after the governance review (F2). This table previously named
`2759908` as phase 5's certifying commit; no CI run exists for it, because it was never a
pushed head — the phase-5 commit that CI actually graded green is `ae7c38d`. Rows 1-3
previously showed no run URL at all while claiming approval was recorded. A certification
table that cannot be checked is not a certification.

- [ ] Gate certified **by the reviewer or user** (not the AI) for every phase above

## Approval

**Decision**: **APPROVED** — relayed by the owner, 2026-09-10. See the provenance note at
the top of this file: no GitHub review exists on any of the four pull requests, so this
line is the record and the owner's word is its authority.

Constitution IX is satisfied by a human approving. It is not satisfied by this file
saying so, if the sentence turns out not to be true — which is why the relay is stated
rather than smoothed over.

## Comments

**This approval does not make the feature mergeable.** Three things are still open, none
of which a human review closes:

1. **Phases 7, 8 and 9 are uncertified.** `Gate Certification: ci-held`, so each needs the
   owner's approval recorded against its evidence triplet in the table above. Phase 8's
   row additionally needs the batch question settled.
2. **The icon rung conflict is undecided** (`tasks.md`, "Phase 9 — rung conflict"). The
   prototype shows no navigation icons but reserves `gap-2.5` for one; `spec.md` VI-005
   names the icon. Constitution II makes that a stop-and-report, and it is still stopped.
   SC-003 asks for an empty deviation table at merge; this is the entry in it.
3. **Constitution 1.1.0 is unmerged** (PR #3). It was written because of this feature and
   applies to it, and merging the feature first would mean the amendments made under it
   were governed by a rule that had not landed.

**For whoever touches this area next.** The most useful artefact here is not the code, it
is `ai-code-review-governance.md` F3: after the single owner approval, five rule changes
were written and used by the same session within minutes, and no machine check noticed,
because the checks grade paths, tokens and dates and none grades authority. Getting the
order right — amend, then implement — is a check on retroactivity, not on consent. The
constitution now says so; nothing yet enforces it.
