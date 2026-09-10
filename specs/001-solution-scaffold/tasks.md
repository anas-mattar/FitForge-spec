# Tasks: Solution Scaffold

**Feature**: `001-solution-scaffold` | **Plan**: `specs/001-solution-scaffold/plan.md`
**Owner**: anas.m | **Reviewer**: ahmad

Territory entries are repo-prefixed and relative to the governance root, per
`docs/sdlc/repository-strategy.md` ("Territory across repositories"). The feature's own
spec directory is implicitly in territory and is never declared.

Each code phase commit lands on the matching `001-solution-scaffold` branch in its code
repository — the Cross-Repository Feature Rule is what lets
`scripts/scope-check-repos.ps1` find this declaration.

## Format: `[ID] [P?] [Story] Description`

`[P]` marks tasks that may run in parallel (different files, no ordering dependency).

---

## Phase 1: API layering and error contract (US2)

**Territory**:

- `fitforge-api/FitForge.slnx`
- `fitforge-api/src/**`
- `fitforge-api/tests/**`
- `fitforge-api/Directory.Build.props`

### Implementation

- [ ] T001 Create `src/FitForge.Domain/FitForge.Domain.csproj` — `net10.0`, nullable enabled, **zero** package references. Add to `FitForge.slnx`.
- [ ] T002 Create `src/FitForge.Infrastructure/FitForge.Infrastructure.csproj` — references `FitForge.Domain` only (EF Core arrives in phase 2). Add to `FitForge.slnx`.
- [ ] T003 Add project references `FitForge.Api → FitForge.Infrastructure` and `FitForge.Api → FitForge.Domain`. Verify no reference points the other way.
- [ ] T004 [P] Create `src/FitForge.Domain/Training/Calculations/` with the module's namespace placeholder and a README-style summary comment naming what belongs here (the `Training.Calculations` module `backend-rules.md` requires). No formulas yet — feature 009 fills it.
- [ ] T005 [P] Add `src/FitForge.Domain/DomainException.cs` — the single base type, plus `NotFoundException` and `DomainRuleViolationException`. The domain names no HTTP status.
- [ ] T006 Delete the `WeatherForecast` endpoint and record from `src/FitForge.Api/Program.cs`.
- [ ] T007 Add `builder.Services.AddProblemDetails()` and a global `IExceptionHandler` in `src/FitForge.Api/Hosting/` (deliberately not `Api/Infrastructure/` — one solution must not use "Infrastructure" for two different things) mapping `NotFoundException` → 404, `DomainRuleViolationException` → 422, everything else → 500. Outside Development the document carries no exception message and no stack trace.
- [ ] T008 Ensure unmatched routes and unmatched methods also produce `application/problem+json` (404 and 405), not framework default HTML.
- [ ] T009 Add `src/FitForge.Api/Features/Health/HealthEndpoints.cs` mapping `GET /health/live` to `{ "status": "live" }`, per `contracts/health.md` §1. It performs no dependency work.
- [ ] T010 Set `TreatWarningsAsErrors` consistently for all projects via `Directory.Build.props` so `--warnaserror` cannot pass by a project simply not having the setting.

### Tests

- [ ] T011 [P] Create `tests/FitForge.Domain.Tests/` (xUnit) with one test asserting the exception hierarchy. Add to `FitForge.slnx`.
- [ ] T012 [P] `tests/FitForge.Api.Tests/HealthLiveTests.cs` — `GET /health/live` returns 200 and the exact contract body.
- [ ] T013 [P] `tests/FitForge.Api.Tests/ProblemDetailsTests.cs` — an unknown route returns 404 with `application/problem+json`; a wrong method returns 405 likewise; a thrown `DomainRuleViolationException` returns 422 and leaks no stack trace.

**Phase exit**: `dotnet build --warnaserror && dotnet test` exits 0. Commit subject carries `phase 1`.

---

## Phase 2: Persistence boundary and readiness (US1 provider half)

**Territory**:

- `fitforge-api/src/**`
- `fitforge-api/tests/**`
- `fitforge-api/appsettings.json`
- `fitforge-api/appsettings.Development.json`

### Implementation

- [ ] T014 Add `Microsoft.EntityFrameworkCore.SqlServer` and `Microsoft.EntityFrameworkCore.Design` to `FitForge.Infrastructure` **only**. Confirm `FitForge.Domain` still has zero package references — this is the ADR's load-bearing fact.
- [ ] T015 Add `src/FitForge.Infrastructure/Persistence/FitForgeDbContext.cs` — no `DbSet` yet, `ApplyConfigurationsFromAssembly` wired so feature 002 only adds a configuration class.
- [ ] T016 Add `src/FitForge.Infrastructure/DependencyInjection.cs` — the single registration extension binding and **validating** database options at startup (`ValidateOnStart`), so a missing connection string fails with a legible message naming the setting rather than an unhandled provider exception (spec, Edge Cases).
- [ ] T017 Add the connection-string **name** to `appsettings.json` with an empty value and document the environment variable in the repository README. No credential enters source.
- [ ] T018 Add health checks: `Microsoft.Extensions.Diagnostics.HealthChecks` plus the SQL Server check, registered so `/health/ready` reports per-dependency results.
- [ ] T019 Map `GET /health/ready` per `contracts/health.md` §1 — 200 with `status: "ready"`, 503 with `status: "degraded"`, `checks[]` carrying `name`, `status`, `durationMs`. The response MUST NOT include server names, connection strings or exception text.

### Tests

- [ ] T020 [P] `tests/FitForge.Api.Tests/HealthReadyTests.cs` — the healthy path returns 200 and the contract shape.
- [ ] T021 [P] Same file — a dependency forced to fail returns 503, `status: "degraded"`, and the response body contains no connection detail (assert the absence explicitly; this is a disclosure test, not a shape test).
- [ ] T022 [P] `tests/FitForge.Api.Tests/ConfigurationTests.cs` — startup with no connection string fails with a message naming the setting.

**Phase exit**: `dotnet build --warnaserror && dotnet test` exits 0 with no SQL Server instance required. Commit subject carries `phase 2`.

---

## Phase 3: Design tokens and app shell (US1 consumer half — UI phase)

**Territory**:

- `fitforge-web/src/**`
- `fitforge-web/package.json`
- `fitforge-web/package-lock.json`
- `fitforge-web/components.json`
- `fitforge-web/postcss.config.mjs`
- `fitforge-web/next.config.ts`

### Implementation

- [ ] T023 Transcribe the prototype's light and dark token sets into `src/app/globals.css` and expose them through Tailwind v4 `@theme`, per plan §4.6. HSL triples and `--radius: 0.65rem` carry over unchanged (VI-011, VI-012).
- [ ] T024 Replace the scaffold's Geist fonts with Inter via `next/font/google`, with a system-UI fallback stack (VI-010).
- [ ] T025 Add the theme-before-paint script to `src/app/layout.tsx` — blocking, stamping the class on `<html>` before first paint (VI-013, spec Edge Cases).
- [ ] T026 Initialise shadcn/ui against those tokens (`components.json`), installing only the prerequisites plan §5 approves.
- [ ] T027 [P] Build `src/components/shell/Sidebar.tsx` — 220px column, bordered card, the five items in fixed order, Profile & settings below a divider (VI-001…VI-005).
- [ ] T028 [P] Build `src/components/shell/BottomNav.tsx` — exactly five items below 1024px; it *replaces* the sidebar rather than hiding it (VI-006).
- [ ] T029 [P] Build `src/components/shell/Header.tsx` — sticky, 1px bottom border, `--card` at 95% with backdrop blur, avatar carrying Profile below 1024px (VI-007).
- [ ] T030 [P] Build `src/components/shell/ThemeToggle.tsx` — the only `"use client"` component in the shell besides the mobile nav.
- [ ] T031 Compose the shell in `src/app/layout.tsx`; replace the Next.js starter content in `src/app/page.tsx` with an empty Today placeholder. The dashboard's content is out of scope (spec, Out of Scope).
- [ ] T032 Add the tabular-figure treatment for numeric text (VI-009) as a utility, not per-component.
- [ ] T033 Grep the diff for colour literals — hex, `rgb(`, named colours — and remove every one (FR-006).

### Visual Compliance Loop

- [ ] T034 Render at ≥1024px and at 375px, in both themes, and compare against `screenshots/fitforge-prototype.html` screen `today`. Produce the deviation table per `docs/sdlc/review-process.md`; iterate until it is empty or every remaining row is user-approved. **This phase does not end when the code compiles.**

**Phase exit**: `npm run lint && npm run typecheck && npm run build && npm test` exits 0 **and** the deviation table is empty or approved. Commit subject carries `phase 3`.

---

## Phase 4: The BFF seam (US1)

**Territory**:

- `fitforge-web/src/**`
- `fitforge-web/.env.example`
- `fitforge-web/package.json`
- `fitforge-web/package-lock.json`

### Implementation

- [ ] T035 Add `src/lib/api-client.ts` with `import "server-only"` at the top — a client component importing it must fail the build, not leak the API address (plan §4.6). It reads `FITFORGE_API_BASE_URL` and applies the contract's 10s timeout with no retry.
- [ ] T036 Add `src/app/api/health/route.ts` implementing `contracts/health.md` §2 — the three-value mapping, always 200, `checkedAt` in UTC ISO 8601.
- [ ] T037 Confirm `.env.example` names `FITFORGE_API_BASE_URL` (already present) and that no value is committed.
- [ ] T038 Wire the health indicator into the shell header, rendering `ready` / `degraded` / `unreachable` distinctly. Fetch from the app's own origin only (US1 scenario 3).
- [ ] T039 Confirm no browser-side code references the API base URL — grep the client bundle output, not just the source.

### Tests

- [ ] T040 [P] `src/app/api/health/__tests__/route.test.ts` — one test per row of the contract's mapping table: 200 → `ready`, 503 → `degraded`, timeout → `unreachable`, connection refused → `unreachable`, unexpected status → `unreachable`.
- [ ] T041 [P] A test asserting the route never returns a non-200 status of its own.

**Phase exit**: `npm run lint && npm run typecheck && npm run build && npm test` exits 0. Commit subject carries `phase 4`.

---

## Phase 5: Binding the checks (US3)

**Territory**:

- `docs/onboarding.md`
- `docs/roadmap.md`
- `kit-adoption.json`

### Implementation

- [ ] T042 Extend `docs/onboarding.md` with the clone-to-running sequence for both repositories — the exact commands, the environment variables to set, and what "working" looks like. SC-001 is measured against this file alone.
- [ ] T043 Wire `ritual-checks` as the single required status check on the governance repository's `main` (`docs/sdlc/branch-protection.md`). **Precondition**: the owner has recorded `gateProof` in `kit-adoption.json`, without which the check is red and would block every pull request.
- [ ] T044 Confirm the code-repository scope check is required on each code repository's `main`.
- [ ] T045 Demonstrate SC-005 once, on a throwaway branch: commit a file outside the declared Territory, observe `scope-check-repos.ps1` FAIL with a non-zero exit code naming the path, then delete the branch. Record the observed output in the phase summary — a check nobody has watched fail is not known to work.
- [ ] T046 Flip the roadmap row for solution-scaffold to `shipped` in a main-side `docs/` commit after merge (the claim-visibility rule cuts both ways).

**Phase exit**: `pwsh -File scripts/ritual-checks.ps1` exits 0 in the governance repository. Commit subject carries `phase 5`.

---

## Dependencies & Execution Order

- **Phase 1 → Phase 2**: phase 2 adds EF Core to projects phase 1 creates.
- **Phase 3 → Phase 4**: the health indicator needs a shell to live in.
- **Phases 1–2 ∥ Phase 3**: different repositories, no shared file. They may be implemented
  in either order or concurrently by the two developers — but each still lands as its own
  phase commit with its own gate.
- **Phase 4 → after Phase 2**: the mapping tests are unit-level and need no running API, but
  the manual end-to-end observation in `contracts/health.md` §4 does.
- **Phase 5 last**: it binds checks over work that must already exist, and T043 is blocked on
  the owner's `gateProof`.

## Notes

- One phase at a time; stop after each and request certification. `**Gate Certification**:
  ci-held` is declared in `plan.md`, so each phase's gate is the owner's recorded approval on
  the evidence triplet — CI run URL, green conclusion, exact phase-commit sha. The agent
  never claims the gate.
- Every phase commit carries a `phase N` token in its subject, or the scope check cannot
  attribute it.
- Territory may be widened only in a governance commit made **before** the code phase commit
  that relies on it. A declaration that post-dates its code FAILs even when every path is
  inside it.

---

## Phase 3 — Visual Compliance Loop result (T034)

Measured in Chrome against `screenshots/fitforge-prototype.html` screen `today`, in both
themes, at 1280px and — via a same-origin iframe, because the window would not resize
below Chrome's minimum — at 357px, which is narrower than the 375px the spec requires.
Computed styles were read rather than eyeballed; "PASS" below means the number matched.

| Item | Verdict |
|---|---|
| VI-001 grid 220px / fluid / 1rem | PASS — `219.99px 884.02px`, gap `16px` |
| VI-002 bordered card, --card, --radius, 0.75rem | PASS — radius `10.4px` (= 0.65rem), padding `12px` |
| VI-003 fixed order, divider before Profile | PASS — one shared list feeds both navigations |
| VI-004 active/inactive colours | PASS — active `rgb(244,244,245)` on `rgb(24,24,27)` weight 500; inactive `rgb(113,113,122)` |
| VI-005 nav padding / radius / size / gap | PASS — `8px 12px`, `8.4px`, `14px`, `10px` |
| VI-006 bottom bar replaces sidebar; 5 items; avatar | **PASS after a fix** — see below |
| VI-007 sticky header, 1px border, 95% + blur | PASS — `sticky`, `blur(8px)`, alpha `0.95` |
| VI-008 40px buttons, primary/secondary | PASS — `Button` primitive, `39.99px`, radius `8.4px` |
| VI-009 tabular figures | PASS — `.num` resolves to `tabular-nums` |
| VI-010 Inter with a real fallback | PASS — `Inter, "Inter Fallback", ui-sans-serif, system-ui, sans-serif` |
| VI-011 light tokens | PASS — exact, incl. `--primary: 22 92% 50%`, `--radius: .65rem` |
| VI-012 dark tokens | PASS — exact, incl. `--primary: 22 92% 54%` |
| VI-013 both themes reachable, survives reload | PASS — class present after reload, `fitforge-theme: dark` |

**The one deviation found and fixed.** At 372px the bottom bar's widest label,
"Programs", cleared its box by **0.4px**. That is not a fit — a 360px device, a larger
default font size, or the fallback face rendering before Inter loads all break it. The
item padding went from `px-2` to `px-1`, giving 5.4px of headroom at 357px with no
wrapping and no horizontal overflow. The reference does not specify the bar's internal
padding, so this departs from nothing.

**Deviation table is empty.** Two items are satisfied but not yet *exercised*, which is
scope, not deviation: only the secondary/icon button variant appears in the shell
(primary and destructive arrive with the screens that need them), and nothing in the
shell displays digits yet, so `.num` is defined and first used by feature 004.
