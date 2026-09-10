# AI Code Review — 001 Solution Scaffold (governance half + cross-cutting coherence)

**Reviewer**: fresh-context agent — claude-opus-5
**Date**: 2026-09-10
**Branches**: FitForge-spec `001-solution-scaffold` (tip `ae7c38d`) · fitforge-api `001-solution-scaffold` (tip `cded3bd`) · fitforge-web `001-solution-scaffold` (tip `8d9232d`)
**Scope reviewed**: the eight governance/spec commits carrying no `phase N` token (`5a34b5d`, `26d9108`, `5e4e8d8`, `8785678`, `7d3f297`, `92455d6`, `be8daef`, `b156260`) plus `3cb6e34` and `cff8c57`; the phase 5 commits `2759908` and `ae7c38d`; `spec.md`, `plan.md`, `tasks.md`, `contracts/health.md`, `human-pr-review.md`; `docs/onboarding.md`, `docs/roadmap.md`, `docs/sdlc/{definition-of-done,gate-command,review-process,branch-protection}.md`, `docs/rulebooks/{backend,frontend,integration}-rules.md`, `.specify/memory/constitution.md`, `kit-adoption.json`; `scripts/{scope-check,scope-check-repos,doc-lint,enforcement-pack}.ps1`; and for cross-repo coherence `fitforge-api/src/**` + `fitforge-api/tests/**` and `fitforge-web/src/**` at their branch tips. Commit-date timelines in all three repositories; live GitHub run and branch-protection state via `gh api`.
**Feature contract**: Standard lane, `**Gate Certification**: ci-held`, `**Gate Batching**: none`; ADR-001 (three projects, `FitForge.Domain` referencing zero packages) is the architecture under constitution IV's bootstrap clause; packages limited to plan §5; no domain entity, no migration; BFF is the browser's only path to the API; every shell item VI-001…VI-013 graded by the Visual Compliance Loop.

## Reviewer Provenance

- **Reviewer**: fresh-context agent — claude-opus-5 (no implementation context; given only the commits and the governance documents)
- **Implementer**: claude-opus-5 (the implementing session; its reasoning was NOT provided to this reviewer)
- **Inputs provided**: all phase and spec commit diffs in three repositories, spec.md, plan.md, tasks.md, contracts/health.md, docs/onboarding.md, docs/roadmap.md, kit-adoption.json, constitution, the sdlc document set
- **Attestation**: This reviewer did not produce the diff under review.

## Verdict

**REQUEST CHANGES** — The engineering is genuinely good: the layering is real and mechanically defended, the tokens are transcribed rather than reinterpreted, the BFF seam is correctly placed and correctly tested, the timestamp discipline the ritual demands was actually kept (every Territory declaration pre-dates the code it governs, verified by committer date, not by commit message), and both gates run green in front of me. What fails is the governance half, and it fails in the specific way governance fails when the same actor writes the rule and the code: **the plan, the tasks and the shared contract were amended five times after the single recorded owner approval, by the implementer, minutes before the code that depended on each amendment — and no verification layer looks at that.** On top of that, **not one AI review exists for six standing phase commits** (DoD gate 5, required at *every* phase commit), **phase 5's nominated certifying commit `2759908` has no CI run and can never have one** (it was never pushed as a head; the two commits that were pushed after it went red), **the feature's own plan declares seven phases and six are delivered**, and one real cross-repo defect survives: a `Degraded` dependency is reported to the browser as **API ready**, with a test whose name claims the opposite. `ritual-checks` prints `RESULT OK` through all of it. The residual risk sits almost entirely in the fact that a green machine verdict on this branch currently means less than it appears to.

## What was verified (evidence)

| Area | Evidence |
|---|---|
| Spec match (FRs implemented as specified) | FR-001…FR-004, FR-007, FR-008, FR-010, FR-011 verified in code (see the SC table below and F4/F7). FR-002: `Program.cs:23-27` (`AddProblemDetails` + `AddExceptionHandler<DomainExceptionHandler>` + `UseStatusCodePages`), asserted by `tests/FitForge.Api.Tests/ProblemDetailsTests.cs`. FR-004: `grep -rn 'DbContext\|SqlConnection\|FromSqlRaw\|ExecuteSqlRaw\|using Microsoft.EntityFrameworkCore' src/FitForge.Api src/FitForge.Domain` → **no matches**; `src/FitForge.Domain/FitForge.Domain.csproj` contains zero `PackageReference` elements. FR-006: `grep -rn 'data-state' fitforge-web/src` → none, and no colour literal found; tokens live only in `globals.css:19-60`. FR-013: blocking script in `src/app/layout.tsx:35` from `theme-script.ts`. **FR-005 not independently re-rendered** — see F19. |
| Visual-reference match: Visual Compliance Loop deviation table attached, empty or user-approved | Table present at `specs/001-solution-scaffold/tasks.md`, "Phase 3 — Visual Compliance Loop result", 13 rows, all PASS, one fixed deviation. Tokens spot-checked against the reference: `screenshots/fitforge-prototype.html:60,65,67,72,77` vs `fitforge-web/src/app/globals.css:23,33,37,45,56` — `--primary: 22 92% 50%` / `22 92% 54%`, `--radius: 0.65rem`, `--success` light+dark all carry over unchanged. **`docs/sdlc/review-process.md:31` also requires the screenshots attached to the phase notes; `ls specs/001-solution-scaffold/screenshots/` returns `fitforge-prototype.html` only** — see F19. |
| Feature contract held (no unapproved table/migration/permission/package) | `fitforge-api`: `FitForge.Infrastructure.csproj` holds exactly the three packages plan §5 (as amended) approves; `FitForge.Api.csproj` holds only `Microsoft.AspNetCore.OpenApi 10.0.12`, inherited from the pre-feature scaffold `8bfd51b` and never named in §5 (noted, not charged). `fitforge-web`: `git show 159d8a0:package.json` vs tip — every added dependency (`class-variance-authority`, `clsx`, `lucide-react`, `tailwind-merge`, `server-only`) is on §5's list; `vitest` predates the feature. No migration exists; `FitForgeDbContext` declares no `DbSet`. **But the approval that legalized the swap is the implementer's own — F3.** |
| Constitution / domain invariants | `modules/training/training-invariants.md` is guarded structurally, as claimed: the zero-package `FitForge.Domain` is real and `Training/Calculations/TrainingCalculations.cs` exists as the named module `backend-rules.md` requires. Constitution I/IV/VII/X breaches recorded as F1, F3, F7, F21. |
| Security (authn/authz, secrets, sensitive logging) | No secret in source: `appsettings.json` carries `Database:ConnectionString: ""` plus a `_comment`; `.env.example` carries names only. Disclosure is genuinely tested — `HealthReadyTests.A_failure_discloses_no_connection_detail` asserts the absence of a planted `"Login failed for user 'sa' on server db-prod-01.internal"` against the **whole** document, and `HealthReadyTimeoutTests` repeats it on the timeout path. `api-client.ts:1` is `import "server-only"`. **Both health endpoints are anonymous and the contract never records that as a decision — F7.** |
| Scope guard (`scope-check.ps1` PASS on the phase commit; `git diff --stat` read for intent) | `pwsh -File scripts/ritual-checks.ps1` → `RESULT OK`, all 7 members, exit 0. `scope-check`: PASS on `2759908`, `be8daef`, `ae7c38d`; two WARNs (`8785678`, `92455d6`) that are correct-by-parent. `scope-repos`: PASS phase 1 `47ace55` (18 files), phase 2 `404a10b` (13), phase 6 `cded3bd` (3), phase 3 `e0df410` (15), phase 4 `8d9232d` (8). **Territory anti-retroactivity verified by timestamp, not by claim** — table in F-note below. Attribution noise recorded as F13; unenforced phase sizing as F11. |
| Rollback safety (phase reverts cleanly; schema additive?) | No schema exists — `FitForgeDbContext` has no `DbSet` and no migration was generated, so there is nothing to roll back at the database. Phases are file-disjoint per repository except phase 6, which deliberately re-opens phase 2's `DependencyInjection.cs`; reverting `cded3bd` restores phase 2 exactly and leaves the gate green (its 3 files are additive plus one edited registration). Phase 5 is documentation only. |

### Territory declaration vs code, by committer date (the rule verified, not assumed)

| Code commit | Repo | Committed | Governing declaration | Declared | Margin |
|---|---|---|---|---|---|
| `47ace55` phase 1 | fitforge-api | 10:09:25 | `5a34b5d` (+`cff8c57` T007 relocation 10:04:05) | 09:48:30 | +20m55s |
| `404a10b` phase 2 | fitforge-api | 10:25:55 | `5a34b5d`; packages by `26d9108` 10:20:34 | 09:48:30 | +37m25s (packages +5m21s) |
| `e0df410` phase 3 | fitforge-web | 10:41:54 | `5a34b5d` | 09:48:30 | +53m24s |
| `8d9232d` phase 4 | fitforge-web | 10:51:45 | `5a34b5d` | 09:48:30 | +63m15s |
| `cded3bd` phase 6 | fitforge-api | 11:15:32 | `8785678` (Territory) 10:52:24; value by `7d3f297` 11:15:03 | 10:52:24 | +23m08s (**value +29s**) |
| `2759908` / `ae7c38d` phase 5 | FitForge-spec | 11:28:36 / 11:32:10 | `5a34b5d` | 09:48:30 | +100m / +104m |

**No declaration post-dates its code.** That rule was kept. The 29-second margin on `7d3f297` is where the rule stops being meaningful — see F3.

### Success criteria SC-001…SC-006, audited independently of `tasks.md`

| SC | Verdict | Evidence |
|---|---|---|
| SC-001 clone → running by `docs/onboarding.md` alone | **NOT MET as written** | Prerequisites table (`docs/onboarding.md:33-37`) omits **git** and **PowerShell 7**, both of which §4 and §5 of the same file require (`pwsh -File scripts/claim-feature.ps1`, `pwsh -File scripts/ritual-checks.ps1`). Also F16's smaller defects. Unverifiable end-to-end today in any case: the shell exists only on `001-solution-scaffold`; `main` in both code repos is still the empty scaffold. |
| SC-002 both gates exit 0 on the merged baseline, certified by the owner | **PARTIAL — technically true, formally uncertified** | I ran them: `dotnet build --warnaserror` → *Build succeeded, 0 Warning(s), 0 Error(s)*; `dotnet test` → 4 + 18 passed, exit 0. `npm run lint` → exit 0; `npm run typecheck` → exit 0; `npm test` → 20 passed / 2 files, exit 0 (`npm run build` not re-run locally; CI's `project-gate` covers it). Green push-event `project-gate` runs exist on **every** phase commit. But "on the merged baseline" is not yet true and phases 5 and 6 are **NOT YET RECORDED** — F2. |
| SC-003 deviation table empty at merge | **NOT MET** | Table is empty only after reclassifying VI-008's primary variant and VI-009's tabular figures as "satisfied but not exercised"; the required screenshots are not attached; the loop was run and graded solely by the implementing agent, whose own review record says "Re-render it yourself". F19. |
| SC-004 zero DB calls outside the boundary, zero API calls from browser code | **MET** | Both greps above return nothing. `api-client.ts` is `server-only`; the browser fetches `/api/health` (`ApiHealthIndicator.tsx:29`), same-origin. |
| SC-005 a deliberately out-of-Territory commit fails the scope check, demonstrated | **NOT MET as specified** | F6: local only, never pushed (`git reflog` in fitforge-api shows `origin/001-solution-scaffold` moving `404a10b → cded3bd`, never touching `9422e6f`), so **CI never ran the failing case** — which is exactly what US3 scenario 2 and US3's Independent Test require. |
| SC-006 feature 002 begins without moving/renaming/re-layering | **LIKELY MET, untestable today** | `Features/<Area>/` exists with one worked example; `ApplyConfigurationsFromAssembly` is wired so 002 adds only a configuration class; `PRIMARY_NAV` is a single list. Genuinely cannot be verified until 002 starts. |

## Findings

### F1 — Six phase commits stand with zero AI reviews, and the machine cannot see it — BLOCKING

`docs/sdlc/definition-of-done.md` is unambiguous: "**Gates 1–5** MUST pass at **every phase commit**", and gate 5 is the fresh-context AI review filed as `specs/NNN-name/ai-code-review*.md`. `docs/sdlc/review-process.md:79` restates it and adds "5. Do not start next phase without approval."

`ls specs/001-solution-scaffold/` returns `contracts/`, `human-pr-review.md`, `plan.md`, `screenshots/`, `spec.md`, `tasks.md`. **There is no `ai-code-review*.md` file of any kind** — this document is the first. Six phase commits (`47ace55`, `404a10b`, `e0df410`, `8d9232d`, `cded3bd`, `2759908`/`ae7c38d`) landed across 79 minutes of committer time with none of them reviewed before the next began.

The reason nothing caught it is structural and worth stating plainly: `scripts/enforcement-pack.ps1:416` only inspects review files that are **ADDED in the branch diff**. Zero reviews added means zero `ReviewProvenance` failures. `ritual-checks` therefore prints `RESULT OK` on a feature with no reviews at all. Gate 5 has no negative check — it verifies the shape of a review that exists and is blind to one that does not.

*Action: implementer — the per-phase obligation cannot be retro-fitted honestly, so record explicitly in `human-pr-review.md` that gates 1–5 were satisfied for phases 1–6 only at feature level (this review plus its code-side sibling), and get the owner's decision on whether that is acceptable for a rehearsal feature. Kit-level: `enforcement-pack.ps1` should fail an `NNN-*` branch that carries phase commits and no `ai-code-review*.md` — file it as a GAP.*

### F2 — Phase 5's nominated certifying commit has no CI run, and two red runs stand in the branch's history — BLOCKING

`human-pr-review.md`, Gate Result table, row 5: commit `2759908`, run "pending push CI". It is not pending; it is impossible. `git reflog` in the governance repository shows `refs/remotes/origin/001-solution-scaffold` moving `7d3f297 → be8daef → b156260 → ae7c38d`. **`2759908` was never a pushed head**, so no workflow ever ran on it. `gh run list -R anas-mattar/FitForge-spec` confirms: head shas `7d3f297`, `be8daef`, `b156260`, `ae7c38d` — no `2759908`.

Worse, the two runs that did fire on the phase-5 push are red:

```
34433529715  ritual-checks  push  be8daef  failure   2026-09-10T03:29:35Z
34433599203  ritual-checks  push  b156260  failure   2026-09-10T03:30:37Z
34433740428  ritual-checks  push  ae7c38d  success   2026-09-10T03:32:48Z
```

So phase 5, whose declared **Phase exit** is "`pwsh -File scripts/ritual-checks.ps1` exits 0 in the governance repository", did not meet its own exit criterion in CI at the commit the review record nominates; it met it two commits later. Under `**Gate Certification**: ci-held`, `docs/sdlc/gate-command.md` requires the run to have "executed on the phase commit itself" and states "A run on any other commit certifies nothing." The only lawful phase-5 triplet is run **34433740428**, conclusion success, sha **`ae7c38d293b91cebe492c15c5301ebbf678d7025`**.

Related and cheap to fix: rows 1–3 of the same table show "—" for the run URL while claiming "recorded 2026-09-10" as owner approval. The runs exist and are green — `34428340737` (`47ace55`), `34429439349` (`404a10b`), `34430480295` (`e0df410`), all push-event `project-gate`, all success. An approval recorded against an uncited triplet is not the triplet the constitution asks for, especially when citing it costs one line.

*Action: implementer — correct the Gate Result table: phase 5's sha becomes `ae7c38d` with run 34433740428; fill the three missing run URLs; note the two red intermediate runs and that `ae7c38d` is the remediation. Owner — re-record approvals against the corrected table.*

### F3 — Every amendment after the single owner approval is self-approved by the implementer, minutes before the code that relies on it — BLOCKING

This is the finding the whole feature turns on, and it is the one the machine is structurally incapable of catching.

The owner approved once: `3cb6e34` at **10:02:57**, stamping `spec.md` `**Status**: Approved 2026-09-10 (owner: anas.m)` and ADR-001 `**Accepted 2026-09-10** by the owner (anas.m)`. After that instant the rules were changed five times, every time by the same agent-driven session that then implemented against them, and **not one carries a recorded approver**:

| Commit | Time | What it changed | Code that relied on it | Gap |
|---|---|---|---|---|
| `cff8c57` | 10:04:05 | T007: host wiring moves to `Api/Hosting/` | `47ace55` 10:09:25 | 5m20s |
| `26d9108` | 10:20:34 | plan §5 package list swapped | `404a10b` 10:25:55 | **5m21s** |
| `8785678` | 10:52:24 | contract §1 gains a new provider MUST; phase 6 created | `cded3bd` 11:15:32 | 23m08s |
| `7d3f297` | 11:15:03 | contract §1 reinterpreted; T047's value 3s → 2s | `cded3bd` 11:15:32 | **29 seconds** |
| `92455d6` | 11:26:26 | phase 7 created | — | — |

Constitution IV: new packages "MUST NOT be introduced unless **explicitly approved in the technical plan**." Plan §5: "Nothing outside this list may be added without amending this plan (constitution IV)." The implementer amended the plan and 5 minutes 21 seconds later shipped against its own amendment. That is not approval; it is a loop with the owner routed out of it. The same shape, tighter, at 29 seconds for the contract reinterpretation that changed a number in the shared law between two repositories.

The amendments are, on their merits, *good*. The package swap is correct (a second `Microsoft.Data.SqlClient` on a .NET 10 stack is a real hazard). Dropping FluentAssertions is correct. The readiness bound is a genuine defect fix. Phase 6 as its own phase rather than a patch is exactly right. **None of that is the point.** The point is that "recorded in a governance commit before the code" was allowed to substitute for "approved", and nothing in `scope-check`, `scope-repos`, `enforcement-pack` or `doc-lint` looks at *who* approved or *whether* they did — those checks grade paths, tokens and dates, never authority. A future feature will use the same loop for an amendment that is not correct on its merits, and the audit trail will look identical to this one.

*Action: owner — before merge, explicitly ratify the four post-approval amendments (`cff8c57`, `26d9108`, `8785678`, `7d3f297`) by re-stamping the Status lines in `plan.md`/`spec.md` and the contract header with a dated approval, or reject any you disagree with. Kit-level: an amendment to `plan.md`/`tasks.md`/`contracts/` on a branch that already has phase commits should require a machine-visible approval marker — file as a GAP; today `scope-check` reads dates and never authority.*

### F4 — A `Degraded` dependency reaches the browser as "API ready", and the test that claims otherwise checks the wrong field — BLOCKING

`contracts/health.md` §1: "**200 OK** — every dependency is usable" and "`status` is `ready` or `degraded` at the top level, and `ready` or `failed` per check."

`fitforge-api/src/FitForge.Api/Features/Health/HealthEndpoints.cs`:

```csharp
Status: report.Status == HealthStatus.Unhealthy ? "degraded" : "ready",
...
Status: entry.Value.Status == HealthStatus.Healthy ? "ready" : "failed",
...
return report.Status == HealthStatus.Unhealthy
    ? Results.Json(response, statusCode: StatusCodes.Status503ServiceUnavailable)
    : Results.Ok(response);
```

ASP.NET Core's `HealthStatus` has three values; the contract has two, and the two collapses are **asymmetric**. A check that reports `HealthStatus.Degraded` produces a document that says `"status": "ready"`, HTTP **200**, and inside it `{"name":"database","status":"failed"}`. The document contradicts itself, "200 OK — every dependency is usable" is false, and the BFF — whose mapping table is correct — reads 200 and tells the developer **"API ready"** while a dependency is down. That is the same collapse the contract's §2 spends a paragraph forbidding, running in the opposite direction and unremarked.

The test named for this passes because it asserts the wrong half:

```csharp
// HealthReadyTests.cs
public async Task A_degraded_dependency_is_not_reported_as_ready()
{
    using var factory = new FitForgeApiFactory { DatabaseStatus = HealthStatus.Degraded };
    ...
    var check = Assert.Single(body.GetProperty("checks").EnumerateArray());
    Assert.Equal("failed", check.GetProperty("status").GetString());
}
```

It never reads `body.status` and never reads `response.StatusCode`. Both are, at that moment, `ready` and `200`. A test whose name states a guarantee it does not assert is worse than no test: it retires the question.

Today the live path is masked — `AddDbContextCheck(..., failureStatus: HealthStatus.Unhealthy)` means the real database check never returns `Degraded`. That masking is one `failureStatus` argument, or one future health check registered without one, away from disappearing, in a file feature 002 will edit.

*Action: implementer — fix in the same phase as the contract change below: either map `report.Status != Healthy` to `"degraded"` + 503, or have the contract state explicitly that `HealthStatus.Degraded` is treated as ready and say why. Then extend `A_degraded_dependency_is_not_reported_as_ready` to assert the document status and the HTTP code, which is what its name promises.*

### F5 — The 3-second bound's stated reasoning does not survive, and "room to spare" is arithmetically false — BLOCKING

`contracts/health.md` §1, added by `8785678`:

> **Readiness MUST answer within 3 seconds** […] This is not a performance target, it is what keeps `degraded` reachable. The consumer applies a 10-second timeout (§3), so a readiness check that takes longer than **that** turns every "the API is degraded" into "the API is unreachable".

"that" is the 10-second timeout. The sentence therefore argues for a bound of **10** seconds and is offered as the justification for **3**. As written the contract's headline requirement has no stated reason at all — the reason given belongs to a different number. Whoever reads §1 next to decide whether 3 is still the right figure will find a paragraph that does not answer the question.

The second paragraph (`7d3f297`) is internally sound — a check bounded at the document's own budget cannot fit inside it — but the measurements that justify the chosen value do not support the conclusion drawn from them. `tasks.md` T047 and `DependencyInjection.cs` both record **2.83s cold** for a document containing **one** 2-second check. That is ~0.83s of overhead around the check and leaves **0.17s** inside a 3-second budget, not the "second of headroom" both documents claim and not "room to spare". And `HealthCheckService` runs registered checks **sequentially**: the moment feature 002 registers a second dependency check with the same 2-second bound, the document's worst case is ~4.8s and blows the contract by 60% — by construction, on the very extension the contract says the headroom is reserved for.

Compounding it: **nothing tests the 3-second bound.** `HealthReadyTimeoutTests` asserts `elapsed < TimeSpan.FromSeconds(10)`, which is the consumer's patience, not the document's budget. The contract's one MUST is unasserted in either repository.

*Action: implementer — rewrite §1's justification so the stated reason matches the stated number; replace "room to spare" with the measured 0.17s; state the per-check budget as a function of the number of registered checks (or make the checks concurrent) rather than as a fixed 2 seconds; and add a test that bounds the whole `/health/ready` document, not one check inside it.*

### F6 — SC-005 was demonstrated locally and never in CI, which is not what US3 asks for — BLOCKING

`spec.md` US3, Independent Test: "**push** a commit that touches a path outside the declared Territory and observe the code-repository scope check fail". US3 Acceptance Scenario 2: "Given a phase commit touching a path outside it, **When CI runs**, Then the scope check fails with a non-zero exit code and names the offending path."

`tasks.md`, "Phase 5 — SC-005 demonstration (T045)": "Made locally, never pushed, reverted immediately afterwards." `git reflog` in fitforge-api confirms it — `origin/001-solution-scaffold` goes `404a10b → cded3bd` and never sees `9422e6f`. The commit survives only as a dangling object (`git fsck --lost-found` → `dangling commit 9422e6ff8f0f…`), reachable to me by luck of a local clone and gone the next `git gc`.

So the thing US3 says must be observed — **CI** failing on a real push — has not been observed by anyone. What was observed is a local script run, transcribed by the implementer into `tasks.md`. T045 itself says "on a throwaway branch"; it was done on the feature branch and `git reset --hard`. The demonstration's own stated purpose — "a check nobody has watched fail is not known to work" — is undercut by the fact that the only witness is the process being audited and the artifact no longer exists in any pushed history.

To be fair to the implementer: the transcript is consistent with the script's real behaviour, and the `code-repo-scope-check` workflow is correctly wired (I read `fitforge-api/.github/workflows/code-repo-scope-check.yml`; it is push-event-only, checks out `anas-mattar/FitForge-spec` unpinned, and runs `-All`). The mechanism is almost certainly sound. It is the *evidence* that does not meet the criterion the feature wrote for itself.

*Action: implementer — push the out-of-Territory commit to a real throwaway branch matching `[0-9][0-9][0-9]-*` in one code repository, let `code-repo-scope-check` go red, cite the failing run URL in `tasks.md`, then delete the branch. That is a five-minute task and it is the difference between SC-005 met and SC-005 asserted.*

### F7 — The contract omits three of the ten elements constitution VII mandates, and the plan ticks VII as satisfied anyway — BLOCKING

Constitution VII: "Each contract MUST define purpose, authentication, endpoints, request schema, response schema, error schema, timeout policy, retry policy, **idempotency strategy**, and **audit requirements**."

`contracts/health.md` defines endpoints, response schemas, an error convention, a timeout policy and a retry policy. It defines **no authentication**, **no idempotency strategy** and **no audit requirements**. `plan.md` Constitution Check ticks VII with "the one integration in this feature — BFF→API — has a **complete** contract at `specs/001-solution-scaffold/contracts/health.md`". Measured against the constitution's own enumeration, it is not complete, and this contract is explicitly designated "the worked example every later contract in `specs/NNN-*/contracts/` is written against" — so the omission propagates by design.

Authentication is not a formality here. Both `/health/live` and `/health/ready` are anonymous. The contract *notices* the exposure ("a readiness probe is reachable by anything that can reach the API") and derives a disclosure rule from it, but never records the access decision itself — while `docs/rulebooks/backend-rules.md` says "anonymous access is opt-in per endpoint and reviewed as a security decision, never a default." The security decision was made and never written down, which is precisely the state VII exists to prevent.

*Action: implementer — add an Authentication section (anonymous, deliberately, with the reasoning already implicit in §1), and one line each for idempotency (GET, safe, idempotent by method) and audit (not audited, and why). Then correct the plan's VII tick, or justify the omission in §7 Complexity Tracking.*

### F8 — A project-wide governance authoring rule was landed inside a feature phase commit, five sections above the rule forbidding exactly that — BLOCKING

`ae7c38d` adds to `docs/onboarding.md` §3:

> One authoring rule for **this file and every other governance document**: a path in backticks must resolve from the governance root […] so write those in bold, not backticks.

`docs/onboarding.md` §8, in the same file:

> A change to the constitution, `docs/sdlc/`, or a rulebook goes on its own `docs/` branch and merges on its own. **Never bundle a rule change with the feature that made you want it — that is how a rule gets adopted without anyone reviewing it as a rule.**

A rule addressed to "every other governance document" is a rule change by any reading, and it rode in on a `phase 5` commit of a feature branch. Nobody will ever review it as a rule; it will be reviewed, if at all, as part of a scaffold feature. That is the exact failure mode §8 names.

Two secondary observations on the rule's content. First, it contradicts itself inside its own sentence — it writes ``fitforge-api`` and ``fitforge-web`` in backticks while instructing the reader not to. (It passes `doc-lint` only because `scripts/doc-lint.ps1:185` requires a `/` before treating a backticked token as a path; the rule's own examples slip through a heuristic gap, not through compliance.) Second, it is a workaround rather than a fix: the durable answer is for `doc-lint` to read `codeRepos` from `kit-adoption.json` and resolve `fitforge-api/...` when the directory is absent from a governance-only checkout. Teaching every future author to degrade cross-repo paths to bold, forever, in a project whose whole topology is multi-repo, is a permanent cost paid for a tooling gap.

*Action: implementer — move the authoring rule out of this feature onto its own `docs/` branch, or get the owner's explicit sign-off that it may ride here; fix its self-contradicting example either way. Separately, file the `doc-lint` + `codeRepos` improvement as a kit GAP.*

### F9 — Merging now merges a feature whose own plan declares seven phases and delivers six — BLOCKING

`plan.md` §6 lists seven phases. Six are committed. Phase 7 ("Make the startup message's own advice work") is fully specified in `tasks.md` with Territory, five tasks and a phase exit, and it is not implemented.

`docs/sdlc/definition-of-done.md`: "The **feature** may be **merged** to `main` only after **its final phase** satisfies items 1–5". By the feature's own plan the final phase is 7. There is no lawful reading of "merge 001 now" that does not first amend `plan.md` §6 and `tasks.md` to remove phase 7 from this feature.

On the substance — is shipping the gap acceptable? I think it is *not*, for a reason narrower than the defect itself. The defect is small: `FitForge.Api.csproj` has no `UserSecretsId`, so `dotnet user-secrets set "Database:ConnectionString" "..."` — the command `DatabaseOptions.MissingConnectionStringMessage` recommends verbatim — exits with *"Could not find the global property 'UserSecretsId'"*. The fix is one csproj property and one test (T051, T055). But the gap is currently documented in **two** places that disagree about the future: `docs/onboarding.md` §2 carries a "Known gap" note pointing at phase 7, and `appsettings.json`'s `_comment` recommends the same broken command with no warning at all. T054 exists to remove the onboarding note when phase 7 lands; nothing removes the `appsettings.json` one. Merging six-of-seven ships an error message that lies, a config comment that repeats the lie, and a governance document that documents the lie — for the sake of not spending ten minutes.

The stronger argument against blocking is that this is a rehearsal feature and the gap is honestly disclosed. I do not find it persuasive: honest disclosure of a defect you have already written the fix tasks for is not a reason to ship the defect.

*Action: owner decides. Recommended: implement phase 7 (T051–T055 plus the `appsettings.json` `_comment`, which T053 should be widened to cover) before merge. Alternative, if the owner wants 001 out: remove phase 7 from `plan.md` §6 and `tasks.md` in an approved governance commit and re-file it as its own Micro feature — do not merge a plan that says seven and a branch that has six.*

### F10 — The runtime "guard" in `DependencyInjection` compares two compile-time constants and defends against an impossible scenario — CONFIRM

T048: "Assert the bound in `DependencyInjection`, not only in configuration, so it cannot be widened past the contract's consumer timeout **by an appsettings edit**."

`fitforge-api/src/FitForge.Infrastructure/DependencyInjection.cs`:

```csharp
if (DatabaseHealthCheckTimeout >= ReadinessConsumerTimeout)
{
    throw new InvalidOperationException(...);
}
```

Both operands are `public static readonly TimeSpan` literals in the same file (`FromSeconds(2)` and `FromSeconds(10)`). No appsettings key feeds either one — the timeout is not configurable at all, so the scenario T048 names cannot occur. The branch is unreachable unless someone edits the source, and someone editing the source sees the constant three lines above. It is enforcement-shaped code that enforces nothing at runtime.

It is also the *wrong* comparison. It guards against exceeding **10s** (the consumer's patience) and not against exceeding **3s** (the contract's document budget) — the bound that actually matters and the one F5 shows is nearly exhausted. `ReadinessDocumentBudget` is declared four lines above and never used outside the test project. A future edit to 9 seconds passes this guard cleanly.

The saving grace is that `HealthReadyTimeoutTests.The_database_check_is_bounded_below_the_consumers_timeout` *does* assert against `ReadinessDocumentBudget`, so the 9-second edit would go red in the gate. The test is doing the work the guard claims to do.

*Action: implementer — either delete the runtime guard and let the test own the invariant, or make it check `< ReadinessDocumentBudget` (the bound that binds) and correct T048's rationale, which is currently false as written.*

### F11 — Constitution X's phase-size guideline is not applied to any code in this feature, and three of five phases exceed it — CONFIRM

`scripts/enforcement-pack.ps1` produced exactly one warning on this branch:

```
PhaseSizeWarning: commit 5a34b5d changes 1780 line(s) across 5 file(s) — exceeds the phase-size guideline (400 lines / 15 files, constitution X).
```

That is the *spec* commit. `Invoke-PhaseSizeWarningCheck` walks the governance branch only; it has no equivalent in `scope-check-repos.ps1`, so the nested code repositories are never sized. Measured by hand:

| Phase | Commit | Files | Lines |
|---|---|---|---|
| 1 | `47ace55` | **18** | **603** |
| 2 | `404a10b` | 13 | **440** |
| 3 | `e0df410` | 15 | **666** |
| 4 | `8d9232d` | 8 | 323 |
| 6 | `cded3bd` | 3 | 226 |

Three of five exceed the guideline; phase 1 exceeds both limits. All of it silently. The guideline is non-blocking by design and a scaffold phase legitimately runs large, so this is not a charge against the phases — it is a charge against the claim that the machine is grading them. In a multi-repo project the phase-sizing check currently covers only the repository that contains no code.

*Action: kit-level GAP — `scope-check-repos.ps1` should emit the same non-blocking `PhaseSizeWarning` for the code repositories' phase commits. Implementer: note the three over-guideline phases in `human-pr-review.md` so the human reviewer knows the machine did not.*

### F12 — `human-pr-review.md` under-reports what is being merged — MINOR

The record was written at `b156260` (11:30:32) and the branch moved twice after it. Its "What merges" table lists `2759908` as the whole of phase 5; `ae7c38d` (also `phase 5`, also in scope-check's PASS list) is absent, as is `b156260` itself from the governance-commit list. Its `scope-check` transcript shows only `PASS phase 5 commit 2759908` where the current run reports three governance PASSes. Combined with F2, a human reviewer reading this table as the merge manifest is reading a manifest two commits stale.

*Action: implementer — regenerate the tables at the branch tip immediately before requesting human review, and add a line stating the tip sha the record describes.*

### F13 — `scope-check` attributes governance prose commits as phase commits by substring, polluting the gate-4 audit trail — MINOR

Phase attribution is `$subject -match '(?i)\bphase\s+(\d+)\b'` (`scripts/scope-check.ps1`). Any commit subject that *mentions* a phase is graded as one. On this branch:

```
scope-check: PASS phase 2 commit 26d9108 (1 file(s))     # "spec: amend plan section 5 package list before phase 2"
scope-check: PASS phase 3 commit 5e4e8d8 (1 file(s))     # "spec: record the phase 3 Visual Compliance Loop result"
scope-check: WARN commit 8785678: no territory declared for phase 6 …
scope-check: WARN commit 92455d6: no territory declared for phase 7 …
scope-check: PASS phase 5 commit be8daef (1 file(s))     # "spec: record the phase 5 … demonstration (T045)"
```

None of those five is a phase commit. `26d9108` — the plan amendment that legalized phase 2's packages — is recorded in the audit trail as *a passing phase-2 scope check*, which is close to the opposite of useful. Both WARNs are technically correct (the declaration is read from the parent, which predates it), but they read as "phase 6 has no territory" when phase 6's territory is declared in the very commit being graded. The verdicts are all harmless here; the trail they produce is misleading in exactly the place a reviewer would go looking.

*Action: kit-level GAP — attribution should require the token in a leading position (e.g. `^phase\s+\d+[:\s]`) rather than anywhere in the subject; the kit's own convention is already `phase N: …`. Meanwhile: implementer, prefer subjects that do not contain a bare "phase N" in prose.*

### F14 — The 10-second consumer timeout is written in three places across two repositories with no link between them — MINOR

- `contracts/health.md` §3: "the BFF applies a 10-second timeout to this call"
- `fitforge-web/src/lib/health.ts`: `export const HEALTH_TIMEOUT_MS = 10_000;`
- `fitforge-api/src/FitForge.Infrastructure/DependencyInjection.cs`: `ReadinessConsumerTimeout = TimeSpan.FromSeconds(10)`, with an honest comment ("The value is owned by the contract, not by this repository")

Each repository asserts the constant against itself (`expect(HEALTH_TIMEOUT_MS).toBe(10_000)`; `registration.Timeout < ReadinessConsumerTimeout`). Neither asserts it against the contract, and nothing asserts them against each other. Change the contract to 5 seconds and both repositories stay green while the API's 2-second check no longer has the relationship the comment describes. The implementer clearly saw this — the comment exists — and had no mechanism to close it, which is fair; it should be named as a standing risk rather than left as a comment.

The inverse problem is also present: `7d3f297` wrote "The database check is bounded at two" into the contract. That is a provider-internal tuning value the consumer cannot observe, now pinned in the shared law — so under §5 ("Changing this contract is a change to both repositories") a purely local API adjustment is formally a cross-repository change. The contract should bound the document and leave the per-check figure to the provider.

*Action: implementer — remove the literal "bounded at two" from the contract in favour of "strictly shorter than the document budget, sized for the number of registered checks"; add a note to §3 that the 10-second figure is duplicated in `fitforge-web/src/lib/health.ts` and `fitforge-api/.../DependencyInjection.cs` and that all three move together.*

### F15 — Frontend rulebook MUSTs are violated, and the checklist meant to enforce them does not exist — CONFIRM

`docs/rulebooks/frontend-rules.md`, UI States: "Every data surface MUST carry a `data-state` attribute naming the active state (`loading` / `empty` / `error` / `ready`), so tests assert the state rather than guessing from rendered text."

`grep -rn 'data-state' fitforge-web/src` → **no matches anywhere.** `ApiHealthIndicator.tsx` is unambiguously a data surface — it holds `useState<ApiHealth | "checking">`, fetches, and renders four distinct states — and carries none. `src/app/page.tsx` renders an empty state and carries none.

The reason it passed: `docs/sdlc/definition-of-done.md` gate 5 says "For phases touching a tier with a compliance checklist (`docs/rulebooks/`), this includes passing that checklist", and each rulebook's own header says "**Binding**: this rulebook is enforced through the [tier]'s compliance checklist". `ls docs/rulebooks/` shows `compliance-checklist-template.md` and **no filled checklist for any tier**. Every rulebook in this project declares itself bound by a document that does not exist. This predates 001, but 001 is the first feature to touch both tiers and is where the hole shows.

Two smaller items in the same family: `ApiHealthIndicator` is `hidden … sm:inline-flex`, so the indicator `docs/onboarding.md` names as the proof both halves are talking is invisible below 640px — on a product whose own rulebook says "this is a phone app used standing at a rack; the desktop layout is the secondary case". And the four nav destinations other than Today (`/library`, `/programs`, `/history`, `/progress`) plus `/profile` have no route and 404; legitimately out of scope per `spec.md`, but unmentioned in the onboarding's "What working looks like".

*Action: implementer — add `data-state` to `ApiHealthIndicator` and `page.tsx` (it is the rulebook's stated mechanism for testing state, and F4's fix will want it); reconsider the `sm:` hide. Owner: author the frontend and backend compliance checklists, or amend the rulebooks to stop claiming a binding mechanism that does not exist.*

### F16 — shadcn/ui was not actually adopted, and the deviation lives only in a commit message — CONFIRM

`plan.md` §4.6: "**shadcn/ui** is installed as the primitive layer, configured against those tokens. It is a code generator, not a runtime dependency — generated components live in `src/components/ui/` and are ours to edit." T026: "Initialise shadcn/ui against those tokens (`components.json`)."

What shipped: `components.json` exists, and `src/components/ui/button.tsx` is **hand-written** — the phase 3 commit message says so plainly ("The Button primitive is hand-written rather than generated because shadcn's takes a Radix dependency purely to support `asChild`, which plan section 5 does not approve"). No shadcn component was ever generated. The same commit message also records that `tw-animate-css` "was installed and then removed" — a package name that appears in neither plan §5 (which approves `tailwindcss-animate`) nor the final `package.json`.

The reasoning is sound and the outcome is arguably better than the plan. That is not the problem. `docs/rulebooks/frontend-rules.md` requires that an edit to a vendored component "MUST be noted in the feature's `plan.md`", and constitution IV makes the plan the place a library decision is approved. Recording a departure from the approved plan **in a commit message** puts it where no source-of-truth rung will ever see it: `plan.md` still tells the next developer that `src/components/ui/` holds generated shadcn components, and the next developer will run `npx shadcn add` against a directory that has a hand-rolled primitive in it.

*Action: implementer — amend `plan.md` §4.6 to state what was actually built (shadcn's config and token wiring adopted, primitives hand-written to avoid an unapproved Radix dependency, no generated component present), with the owner's approval per F3.*

### F17 — `docs/onboarding.md` has gaps that SC-001 measures against this file alone — CONFIRM

Read as a developer with nobody to ask, following it top to bottom:

1. **Prerequisites omit `git` and PowerShell 7.** The table lists .NET SDK, Node and SQL Server. §4 opens with `pwsh -File scripts/claim-feature.ps1` and §5 step 3 with `pwsh -File scripts/ritual-checks.ps1`. `pwsh` is not `powershell` and is not installed by default on Windows. Verified present here (7.6.6) but never stated as a requirement.
2. **The verification `curl` contradicts the section three screens later.** `curl -i http://localhost:5212/health/ready # 200 + {"status":"ready",…}` is presented unconditionally, immediately after connection-string setup. "Running without a database" then tells the developer to point the string nowhere — in which case that same curl returns **503**. A developer taking the no-database path hits an apparent failure at the one step the document told them to use as proof of success.
3. **"Give the API any syntactically valid connection string pointing nowhere" supplies no example.** Every other command in the document is copy-pasteable; this one asks the reader to invent a value, and the obvious move (reuse the `Server=localhost` string above) may connect on a machine that has SQL Server.
4. **Launch-profile dependence is unstated.** `dotnet run --project src/FitForge.Api` selects the first profile in `launchSettings.json`, `http`, which is why `http://localhost:5212` works and why `app.UseHttpsRedirection()` is inert. Run under the `https` profile (`applicationUrl: "https://localhost:7016;http://localhost:5212"`) and the documented curl 307s and `FITFORGE_API_BASE_URL=http://localhost:5212` starts redirecting. One sentence fixes it.
5. **"What working looks like" over-promises.** The health indicator is hidden below 640px (F15); four of the five nav items 404.

Credit where due: the Known-gap note, the Windows socket-10013 row, the `Invalid framework identifier ''` XML-comment row and the three-word health vocabulary table are unusually good, and are clearly the product of actually running the file rather than describing it. The gaps above are the residue of running it on one machine.

*Action: implementer — add git and PowerShell 7 to the prerequisites; make the readiness curl conditional or move it after "Running without a database"; supply a literal points-nowhere connection string; add one sentence on the launch profile.*

### F18 — The contract's "agreed before implementation" header is no longer true, and §5's own change policy was not followed — DOC DRIFT

`contracts/health.md` header: "**Status**: Agreed (constitution VII — this contract precedes implementation on both sides)."

§1's readiness bound was added at **10:52:24** (`8785678`) and reinterpreted at **11:15:03** (`7d3f297`) — after the provider phases (`47ace55` 10:09, `404a10b` 10:25) and after **both** consumer phases (`e0df410` 10:41, `8d9232d` 10:51). For that clause the header states the opposite of what happened. §5 further says "Changing this contract is a change to **both** repositories"; this change produced a commit in one.

The amendment itself is defensible — it is a defect fix discovered by the very end-to-end observation §4 mandates, and it was recorded before the code that implemented it. But an unversioned contract whose header claims a property it no longer has is the thing that makes contracts stop being believed. This document is also the designated worked example for every later contract in the project.

*Action: implementer — give the contract a version/amendment log ("v1.0 agreed 2026-09-10 before implementation; v1.1 §1 readiness bound added 2026-09-10 after phase 4's end-to-end observation, owner-approved <date>"), and reword the header so "Agreed" no longer asserts something false about §1.*

### F19 — SC-003's empty deviation table is reached partly by reclassification, and the required screenshots are absent — CONFIRM

`docs/sdlc/review-process.md:31`: "Attach the final table and **both screenshots** to the phase notes — the AI review verifies they exist." `ls specs/001-solution-scaffold/screenshots/` returns `fitforge-prototype.html` alone. There is no captured render of the implementation, at either viewport, in either theme. I am the AI review that was supposed to verify they exist; they do not, so I cannot independently confirm a single VI row. Everything in that table is the implementer's measurement of its own output.

Second, two rows are marked PASS on evidence that does not support PASS. The table's own closing note says the primary Button variant and the `.num` tabular treatment "are satisfied but not yet *exercised*" — VI-008 measured only the secondary/icon variant, and VI-009 measured a CSS rule that nothing in the shell applies. Calling that "scope, not deviation" is a reasonable position for the owner to accept; recording it as PASS inside a table whose emptiness is the SC-003 criterion is not the same thing.

Third, and in the implementation's favour: the loop was run at **357px** rather than the specified 375px, which is *stricter*, and the one deviation found (a 0.4px overflow on "Programs") is the kind of thing only a genuine computed-style measurement catches. This was not a rubber stamp. It is simply not independently verifiable, which is what SC-003 needs it to be.

*Action: implementer — attach the four captures (≥1024px and 375px × light and dark) to `specs/001-solution-scaffold/screenshots/`; re-mark VI-008 and VI-009 as "deferred, not exercised in this feature" with owner acceptance rather than PASS. Human reviewer — re-render, as `human-pr-review.md` already asks.*

### F20 — The BFF maps "not configured" onto "unreachable", a case the contract does not cover — MINOR

`fitforge-web/src/app/api/health/route.ts` catches the `getApiHealth()` throw (raised only when `FITFORGE_API_BASE_URL` is unset) and returns `{ api: "unreachable" }`. The contract's §2 mapping table has three rows — 200, 503, and "timeout, connection refused, DNS failure, any non-200/503" — and no row for "we never tried". The code comment is honest about this ("the contract has no fourth word for 'we never tried'"), but the effect is that a **deployment misconfiguration** presents identically to an **API outage**, which is the same class of conflation §2 spends a paragraph forbidding. `docs/onboarding.md`'s troubleshooting table partly rescues it ("API unreachable → the API process, **or** `FITFORGE_API_BASE_URL`"), so a developer following the docs will get there; a developer reading only the contract will not.

*Action: implementer — add a row to §2's table for "consumer not configured → `unreachable`", or introduce a fourth value. A one-line contract addition either way; the current code is fine once the contract says so.*

### F21 — `docs/sdlc/gate-command.md` contradicts the recorded adoption state — DOC DRIFT

`docs/sdlc/gate-command.md`: "**Proof status — agent-run, awaiting the owner's confirmation.** […] The `gateProof` array in `kit-adoption.json` stays empty — and the adoption doctor stays red on it — until the owner runs both chains and records the exit codes themselves."

`kit-adoption.json` carries two `gateProof` entries recorded by `anas.m` on 2026-09-10, and `verify-kit` reports `ok record — … gate proven`. The document describes a state that ended at commit `7687dc1`. It sits one rung below the constitution in the source-of-truth ladder and currently tells a reader the project's gate is unproven.

`kit-adoption.json` is inside phase 5's declared Territory and was not touched; T043's stated precondition ("the owner has recorded `gateProof`") was satisfied during adoption, and `gate-command.md` was simply never brought forward.

*Action: implementer — update the Proof status block to record the owner's confirmed exit codes and dates from `kit-adoption.json`. Per F8 this is a `docs/sdlc/` change and belongs on its own `docs/` branch.*

### F22 — Two of phase 2's declared Territory entries name files that do not exist, and nothing validates that — MINOR

`tasks.md`, Phase 2 **Territory**:

```
- `fitforge-api/src/**`
- `fitforge-api/tests/**`
- `fitforge-api/appsettings.json`
- `fitforge-api/appsettings.Development.json`
```

Neither of the last two exists. `ls fitforge-api/appsettings.json` → *No such file or directory*; the real files are `fitforge-api/src/FitForge.Api/appsettings.json` and `…/appsettings.Development.json`, already covered by the `fitforge-api/src/**` entry above. T017 landed in the right place; the declaration simply names the wrong one.

Harmless in effect — the phase passed on the broader glob — but instructive about what gate 4 actually checks. `scope-check-repos.ps1` does validate that a territory entry's **first segment** is a declared code repository (its typo guard, `Test-DeclaredRepos`), and it validates that every changed path is *inside* the territory. It never validates that a declared entry corresponds to anything real. Two dead entries sat inside a declared Territory across five graded phase commits and produced `PASS` every time. A Territory is a promise about what a phase may touch; an entry naming a nonexistent path silently widens the promise's *apparent* precision without widening its coverage, and a reviewer reading the declaration would reasonably believe phase 2 was authorised to edit a repository-root config file it cannot reach.

*Action: implementer — delete the two entries (they are redundant) or correct them to `fitforge-api/src/FitForge.Api/appsettings*.json`, in a governance commit; no phase re-commit is needed since no verdict changes. Kit-level GAP: both scope checkers should emit a non-blocking WARN for a territory entry that matches no path in the repository at the graded commit — a declaration that names nothing is a declaration nobody can review.*

### F23 — Verified and correct: things I went looking for and did not find — ACCEPTED

Recorded so the human reviewer does not re-spend the time:

- **No back-declared Territory.** Every declaration pre-dates its code by committer date (table above). `scope-check-repos.ps1` reads the declaration via `git rev-list -1 --before="$When" $GovRef -- $RelPath`, i.e. by committer date with a path filter, and its verdict here is honest.
- **Branch protection matches T043/T044's claim.** `gh api` on all three repositories: `FitForge-spec` requires `ritual-checks`; `fitforge-api` and `fitforge-web` each require `project-gate` + `code-repo-scope-check`; `strict: true` and `enforce_admins: true` everywhere; `allow_force_pushes` and `allow_deletions` false. `required_pull_request_reviews` is null in all three, which matches what `docs/sdlc/branch-protection.md:52` prescribes — so constitution IX is enforced by convention, not by the platform. That is the kit's choice, not this feature's defect, but it is worth the human reviewer knowing before merging.
- **Green push-event `project-gate` runs exist on every code phase commit** (34428340737, 34429439349, 34430480295, 34431163998, 34432663594) — the evidence for F2's fix is available, not missing.
- **No secret, no colour literal, no database access outside the boundary, no browser-side API address.** All four verified by search.
- **`bin/`/`obj/` are not tracked** in `fitforge-api` (`git ls-files | grep -E '(bin|obj)/'` → empty), despite being present on disk.
- **Both gates are genuinely green**, run by me as feedback only: `dotnet build --warnaserror` 0 warnings / 0 errors, `dotnet test` 4 + 18 passed; `npm run lint`, `npm run typecheck`, `npm test` (20 passed) all exit 0.

*Action: none.*

## Constitution re-check (post-implementation)

**FAIL** — four principles are engaged and not satisfied as the plan claims.

| Principle | Plan claim | Re-check |
|---|---|---|
| **I. Specification First** | satisfied — spec/plan/tasks approved before any code phase | **PARTIAL.** True at 10:02:57 and not maintained: four subsequent amendments carry no approval (F3). |
| **II. Source of Truth** | no conflict | **ENGAGED-LATE.** Rung 5 (the contract) was edited after both sides implemented against it, and its header still asserts otherwise (F18). No rung inversion found in the code itself — the prototype's tokens are transcribed exactly. |
| **III. Repository Separation** | satisfied | **PASS.** No phase touched application code in both repositories; phase 5 is governance-only (`scope-check`: 1 file). |
| **IV. Architecture Consistency** | satisfied — packages enumerated in §5 and approved here | **FAIL as a control, PASS as an outcome.** No unapproved package shipped (verified against `159d8a0`/`8bfd51b`), but the approval that legalized the swap is the implementer's own (F3), and the delivered frontend departs from §4.6's shadcn decision with the departure recorded only in a commit message (F16). |
| **V. Domain Invariants** | none implementable | **PASS.** Structural guard is real: `FitForge.Domain.csproj` has zero `PackageReference`, `Training/Calculations/` exists. |
| **VI. Security** | satisfied — nothing protected exists yet | **PARTIAL.** Secrets discipline is clean and disclosure is properly tested. But two anonymous endpoints ship with no recorded access decision, which `backend-rules.md` requires and VII mandates (F7). |
| **VII. External Integration Governance** | satisfied — "a complete contract" | **FAIL.** Authentication, idempotency strategy and audit requirements are absent — 3 of the 10 elements VII enumerates — in the document explicitly designated the project's worked example (F7). |
| **VIII. Testing Requirements** | coverage for the readiness failure path and every mapping row | **PASS with one hole.** Both are covered (see below). The contract's one MUST — the 3-second document bound — is asserted nowhere (F5). |
| **IX. Human Review** | Ahmad reviews before merge | **PENDING.** `gh pr list` shows all three PRs OPEN with empty `reviewDecision`. Correct at this stage. |
| **X. Controlled Delivery** | five phases, one at a time, ci-held declared before the first phase | **FAIL.** Seven phases now declared, six delivered (F9); no AI review gated any phase transition (F1); phase 5's certifying evidence does not exist as recorded (F2); phase sizing was never applied to any code (F11). `ci-held` was correctly declared before the first phase, and the agent consistently and correctly refused to claim the gate in every commit message — that part of X was honoured well. |

## Test coverage observed

**fitforge-api — 22 tests, all passing** (`dotnet test`, Debug):

- `FitForge.Domain.Tests` (4) — `DomainExceptionTests`: the exception hierarchy, one base type with `NotFoundException` / `DomainRuleViolationException` beneath it. Thin by design; the project has nothing else in it yet.
- `FitForge.Api.Tests` (18):
  - `HealthLiveTests` — 200 and the exact contract body for `/health/live`.
  - `ProblemDetailsTests` — unknown route → 404 `application/problem+json`; wrong method → 405 likewise; thrown `DomainRuleViolationException` → 422 with no stack trace. This is the FR-002 "no framework default HTML is reachable" claim, actually asserted.
  - `HealthReadyTests` (4) — healthy → 200 + shape; unhealthy → 503 + `degraded` + `failed`; **`A_degraded_dependency_is_not_reported_as_ready` — asserts only the per-check field and therefore does not test its own name (F4)**; `A_failure_discloses_no_connection_detail` — the strongest test in the suite: plants `"Login failed for user 'sa' on server db-prod-01.internal"` and asserts its absence from the **whole raw document**, not from a named property. That is the right instinct.
  - `HealthReadyTimeoutTests` (3) — the registered check carries a real timeout (explicitly ruling out `Timeout.InfiniteTimeSpan`, which would pass a naive `<` comparison while meaning the opposite — a genuinely good catch); the timeout is `< ReadinessConsumerTimeout` **and** `< ReadinessDocumentBudget`; an overrunning check still yields 503 + `degraded` rather than 500 or a hang; the timeout path discloses nothing. `ResolveDatabaseRegistration()` deliberately builds a fresh `ServiceCollection` rather than reading the test host, "that one replaces the registrations, so asserting against it would prove only that the stub is correct" — correct reasoning, and the kind of thing that is usually got wrong.

**fitforge-web — 20 tests, 2 files, all passing** (`vitest run`):

- `src/lib/__tests__/health.test.ts` — one test per row of the contract's §2 mapping table (200→ready, 503→degraded, timeout→unreachable, ECONNREFUSED→unreachable, ENOTFOUND→unreachable, plus a parameterised sweep of 500/502/404/401/301), an explicit `classifyStatus(503) !== classifyStatus(500)` non-collapse assertion, `checkedAt` is UTC `Z`-suffixed against a frozen clock, fetch called **exactly once** (the no-retry rule), the trailing-slash base URL case, and "never throws whatever the transport does". This is the best-tested surface in the feature and it is tested at the right level — the mapping was deliberately extracted into a pure module so it could be.
- `src/lib/__tests__/units.test.ts` — pre-existing.

**Not covered anywhere:** the readiness *document's* 3-second bound (F5); the `HealthStatus.Degraded` → document-status path (F4); `route.ts`'s unconfigured-base-URL branch (F20); any rendering test of the shell (no component tests exist, and no captured screenshots — F19). Contract §4's manual end-to-end was performed and recorded in `8d9232d`'s message with real numbers, which is how it found the phase-6 defect; it has not been re-run since phase 6 landed, and phase 6's own exit criterion requires exactly that ("a manual re-run of phase 4's cold observation reports `degraded` rather than `unreachable`") — I found no record that it was.

## Residual risk

The code risk is low and well-contained. The governance risk is the whole of it, and it concentrates in one place: **on this branch, a green `ritual-checks` means less than it looks like it means.** It certifies that paths stayed inside declared territories, that declarations pre-dated code, that documents' backticked paths resolve, that digests match their law and that the roadmap is honest. It certified all of that correctly. It is simultaneously blind to a feature with zero AI reviews (F1), to a plan amended five times by the party it constrains (F3), to a phase whose certifying commit has no CI run (F2), and to three of five phases exceeding the size guideline (F11). Feature 001 is the project's rehearsal; whatever it establishes as "what a green branch means" is what features 002 through 011 will inherit.

Carried by which findings, in order of what I would not merge without:

1. **F3** — the self-approval loop. Everything else is a bug; this is a *mechanism*, and it will be reused. Fix it here or it becomes the project's normal.
2. **F1** — six unreviewed phase commits, with no machine check capable of noticing. Also a mechanism.
3. **F4** — the one live cross-repo defect: a degraded dependency reads as "API ready" in the browser, guarded by a test whose name says otherwise. Currently masked by a single `failureStatus` argument in a file feature 002 will edit.
4. **F2**, **F9**, **F6**, **F7**, **F5**, **F8** — each individually cheap to close, and each one a place where the recorded state and the actual state differ.

**Merge only after**: F3's amendments are ratified by the owner; F9 is resolved (phase 7 implemented, or removed from the plan by an approved commit); F2's Gate Result table is corrected and re-approved against real run URLs; F4's mapping and its test are fixed; F5, F6, F7 and F8 are closed. F10, F14, F15, F16, F17, F18, F19, F20, F21 and F22 can follow merge if the owner accepts them explicitly and each gets a home — but F19's screenshots should land before the human review, because without them no one but the implementer has ever seen this UI.
