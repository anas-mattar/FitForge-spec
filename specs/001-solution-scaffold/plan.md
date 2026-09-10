# Implementation Plan: Solution Scaffold

**Branch**: `001-solution-scaffold` | **Date**: 2026-09-10 | **Spec**: `specs/001-solution-scaffold/spec.md`
**Input**: Feature specification from `/specs/001-solution-scaffold/spec.md`
**Gate Batching**: none
**Gate Certification**: ci-held

## Summary

Reshape the two tool-default scaffolds into FitForge's baseline: a layered C# API that
returns RFC 9457 problems and answers liveness and readiness separately, a Next.js
application carrying the prototype's design tokens and app shell, and a BFF route that is
the browser's only path to the API. Five phases, each independently revertible.

The centre of this plan is the ADR in §4. Per constitution IV's bootstrap clause, that ADR
**is** FitForge's architecture until this feature merges; afterwards it is "the existing
architecture" every later feature must follow, and `docs/rulebooks/backend-rules.md` already
defers to it by name.

## Technical Context

**Language/Version**: C# / .NET 10 (SDK 10.0.202, pinned in `global.json`); TypeScript 5 on Node 22
**Primary Dependencies**: ASP.NET Core Minimal APIs, EF Core 10, Next.js 16.3.4 (App Router, Turbopack), React 19.2, Tailwind CSS v4, shadcn/ui
**Storage**: SQL Server. Structural wiring only in this feature — no entity, no migration applied
**Testing**: xUnit + `WebApplicationFactory` in **fitforge-api**; Vitest in **fitforge-web**
**Target Platform**: Linux/Windows server for the API; Node server for the web application
**Project Type**: Web application across two repositories (`fitforge-api`, `fitforge-web`)
**Performance Goals**: none set for a scaffold beyond "the health probe answers in well under its 10s timeout"
**Constraints**: the gate command in each repository exits 0 at every phase boundary; `--warnaserror` stays on
**Scale/Scope**: 2 developers, 11 planned features, 11 screens in the visual reference

## Constitution Check

*GATE: passed before Phase 0. Re-check after the ADR is approved.*

- [x] **Specification First (I)**: `spec.md`, this plan and `tasks.md` land and are approved before any code phase.
- [x] **Source of Truth (II)**: no conflict. The prototype is rung 2 and this plan transcribes it rather than reinterpreting it; where Tailwind v4 cannot express the prototype's v3 CDN config literally, §4.6 records the translation instead of silently diverging.
- [x] **Repository Separation (III)**: backend code is confined to `fitforge-api`, frontend and BFF code to `fitforge-web`. No phase touches both repositories except phase 5, which touches only governance documents and CI configuration — never application code in both.
- [x] **Architecture Consistency (IV)**: this feature *establishes* the architecture under the bootstrap clause. New packages are enumerated in §5 and approved here; anything not listed is out of scope.
- [x] **Domain Invariants (V)**: no domain rule is implemented here, so none can be violated. The ADR is nonetheless chosen so that invariants 1 (immutable completed sessions) and 5 (derived, never stored) have a pure layer to live in — see §4.3.
- [x] **Security (VI)**: no authentication ships (spec, Out of Scope), and nothing protected exists yet to protect. No secret enters source: `FITFORGE_API_BASE_URL` and `FITFORGE_SESSION_SECRET` are names in `.env.example` and values in the environment. The readiness probe discloses no connection detail (contract §1).
- [x] **External Integration Governance (VII)**: the one integration in this feature — BFF→API — has a complete contract at `specs/001-solution-scaffold/contracts/health.md`, agreed before either side is implemented.
- [x] **Testing Requirements (VIII)**: no business-critical logic ships. Coverage is still required for the two things that *are* logic here: the readiness check's failure path and every row of the BFF's reachability mapping table.
- [x] **Human Review (IX)**: Ahmad reviews before merge, per the roadmap's cross-review rule. This is Anas's feature.
- [x] **Controlled Delivery (X)**: five phases, one at a time, each independently revertible. `ci-held` is declared above, before the first phase — each phase's gate is satisfied by the owner's recorded approval on the evidence triplet (CI run URL + green conclusion + exact phase-commit sha).

**Phase sizing self-check**: phases 1 and 2 are both API-side but are not one phase — phase 1 has no dependency and no configuration requirement, while phase 2 introduces both. Reverting phase 2 leaves phase 1 correct and the gate green. Phases 3 and 4 are separated on the same test: the shell renders and is gradeable against the Visual Inventory with no BFF at all.

## Project Structure

### Documentation (this feature)

```text
specs/001-solution-scaffold/
├── spec.md
├── plan.md                              # this file — contains the architecture ADR
├── tasks.md
├── contracts/health.md                  # BFF <-> API health contract
└── screenshots/fitforge-prototype.html  # visual reference, frozen for this feature
```

No `data-model.md`: the feature introduces no entity (spec, Key Entities). The first one
arrives with feature 002. No `research.md`: the two open questions (C# layering, Tailwind v4
token translation) are decided in §4 rather than explored separately.

### Source Code

```text
fitforge-api/
├── FitForge.slnx
├── global.json
├── src/
│   ├── FitForge.Domain/          # NEW — pure. No EF Core, no ASP.NET Core.
│   ├── FitForge.Infrastructure/  # NEW — EF Core: DbContext, configurations, migrations.
│   └── FitForge.Api/             # host, endpoints as Features/<Area>/, DI composition
└── tests/
    ├── FitForge.Domain.Tests/    # NEW — pure unit tests, no host
    └── FitForge.Api.Tests/       # WebApplicationFactory integration tests

fitforge-web/
└── src/
    ├── app/
    │   ├── layout.tsx            # shell: header, sidebar, bottom bar, theme script
    │   ├── globals.css           # design tokens (@theme) transcribed from the prototype
    │   └── api/health/route.ts   # the BFF route — the browser's only path to the API
    ├── components/ui/            # shadcn/ui primitives
    ├── components/shell/         # sidebar, bottom tab bar, header, theme toggle
    └── lib/
        ├── api-client.ts         # server-only typed client for the C# API
        └── units.ts              # already present
```

## 4. ADR-001 — FitForge API architecture

**Status**: **Accepted 2026-09-10** by the owner (anas.m). Under constitution IV's bootstrap
clause this ADR is FitForge's architecture from now until 001 merges; afterwards it is "the
existing architecture" every later feature must follow.
**Context**: FitForge's domain is small in surface but has real, stated invariants — a
completed session is immutable and corrected only additively; personal records, volume,
streak and adherence are *derived and never stored*. Those two rules are exactly the ones
that decay when calculation code sits next to an ORM and someone finds it convenient to
persist a computed total. Two developers own whole vertical slices, so ceremony that
requires a fourth file per feature will be abandoned within a month, and abandoned structure
is worse than no structure.

### 4.1 Options considered

1. **Single project.** Everything in `FitForge.Api`. Cheapest to start; nothing prevents
   `Training.Calculations` from taking a `DbContext`, and nothing prevents a derived value
   from being persisted "just for performance". Rejected: it removes the only mechanical
   protection the two invariants above can have.
2. **Two projects** (`Api` + `Domain`, EF inside `Api`). Better, but migrations and mappings
   then live beside HTTP concerns and the persistence boundary is a convention rather than a
   compile-time fact. Rejected.
3. **Four-project Clean Architecture** (`Domain`, `Application`, `Infrastructure`, `Api`),
   typically with MediatR. Rejected: the `Application` layer earns its keep when several
   hosts share it. FitForge has one host. It would add a package, an indirection and a file
   per operation for no benefit at this size — and constitution IV means every later feature
   inherits that cost.
4. **Three projects** (`Domain`, `Infrastructure`, `Api`), with application logic as vertical
   feature folders inside `Api`. **Chosen.**

### 4.2 Decision

Three projects, dependencies pointing one way only:

```text
FitForge.Api  ──►  FitForge.Infrastructure  ──►  FitForge.Domain  ──►  (nothing)
      └──────────────────────────────────────────►
```

- **`FitForge.Domain`** — entities, value objects, domain exceptions, and
  `Training.Calculations` (the single named calculation module `backend-rules.md` requires).
  It references **no** package: no EF Core, no ASP.NET Core, no JSON library. This is the
  decision that carries the ADR; everything else follows from it.
- **`FitForge.Infrastructure`** — `FitForgeDbContext`, `IEntityTypeConfiguration<>` mappings,
  migrations, and the DI registration extension that binds and validates database options.
  References `Domain`.
- **`FitForge.Api`** — the host. Endpoints live in `Features/<Area>/` folders, one folder per
  slice, holding the endpoint mapping, its request/response contracts and its handler
  together. References `Infrastructure` and `Domain`.

### 4.3 The persistence boundary, stated precisely

`FitForge.Infrastructure` is the boundary. Concretely:

- `FitForgeDbContext`, every entity configuration and every migration live there and nowhere
  else.
- `FitForge.Domain` MUST NOT reference EF Core. Enforced by the absence of the package
  reference — adding it is a visible, reviewable diff, which is the point.
- Raw SQL and ADO.NET MUST NOT appear outside `FitForge.Infrastructure`.
- Feature handlers in `FitForge.Api` MAY inject `FitForgeDbContext` and query through it.
  There is no repository abstraction: `DbSet<T>` already is one, and wrapping it would be
  option 3's ceremony under another name. Reads use `AsNoTracking()`
  (`backend-rules.md`).

The boundary is therefore *configuration and mapping*, not *every line that touches data* —
stated this way because it is checkable, and a rule nobody can check is a rule nobody keeps.

### 4.4 Error handling

- `AddProblemDetails()` plus a global `IExceptionHandler`. Every failure — 404 and 405
  included — leaves the API as `application/problem+json` (RFC 9457). No framework default
  HTML page is reachable.
- One domain exception base type in `FitForge.Domain`, mapped in `FitForge.Api` to 422 for a
  violated rule and 404 for a missing resource. The domain never names an HTTP status; the
  Api layer never re-implements a rule.
- Problem documents carry no exception message or stack trace outside Development.

### 4.5 Consequences

- Adding a feature means adding one folder under `Features/`, plus whatever the domain needs
  — the cheap path and the correct path are the same path, which is the only way structure
  survives contact with a deadline.
- A calculation cannot reach the database without someone adding a package reference to a
  project that has none. That is the mechanical protection invariants 1 and 5 need.
- If FitForge ever grows a second host (a worker, a scheduled job), extracting an
  `Application` project from the `Features/` folders is a mechanical move. We are not paying
  for that today.
- Cost accepted: the `Api` project holds application logic, so it is not a pure delivery
  layer. Reviewers must watch for domain rules drifting into handlers — the specific risk
  this shape carries, named here so review can look for it.

### 4.6 Frontend and BFF decisions

- **Server components by default.** A component becomes `"use client"` only where interaction
  demands it (the theme toggle, the mobile nav). The shell itself renders on the server.
- **Tokens, not values.** The prototype's `:root` / dark token sets are transcribed literally
  into `globals.css` and exposed to Tailwind through v4's `@theme`. *Translation recorded*:
  the prototype uses a Tailwind v3 Play-CDN `theme.extend.colors` block, which v4 replaces
  with `@theme`; the HSL triples and `--radius: 0.65rem` carry over unchanged, so the output
  is identical and only the declaration site moves. No component may write a colour literal.
- **shadcn/ui** is installed as the primitive layer, configured against those tokens. It is a
  code generator, not a runtime dependency — generated components live in
  `src/components/ui/` and are ours to edit.
- **Inter via `next/font/google`**, self-hosted at build time. The prototype loads Inter from
  a CDN because a single HTML file must; the application must not add a third-party request
  to every page load to achieve the same face.
- **Theme before paint**: a tiny blocking script in `layout.tsx` reads the stored preference
  and stamps the class on `<html>` before first paint. Deliberately not a client component —
  by the time React hydrates, the flash has already happened.
- **The BFF is not a tier that owns anything.** `src/app/api/health/route.ts` maps the
  contract and nothing else. `FITFORGE_API_BASE_URL` is read server-side only; `api-client.ts`
  carries `import "server-only"` so that a client component importing it fails the build
  rather than leaking the API's address into the bundle.

## 5. Packages approved by this plan

Nothing outside this list may be added without amending this plan (constitution IV).

**fitforge-api**: `Microsoft.EntityFrameworkCore.SqlServer`, `Microsoft.EntityFrameworkCore.Design`, `Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore`; test-side `xunit`, `Microsoft.AspNetCore.Mvc.Testing`.

*Amended 2026-09-10, before the phase 2 commit.* Two changes, both narrowing:

- `AspNetCore.HealthChecks.SqlServer` → `Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore`. The third-party package's newest release is 9.0.0 and carries its own `Microsoft.Data.SqlClient`, which on a .NET 10 stack means a second SQL client version resolved against EF Core 10's. The first-party package ships 10.0.12, aligned with everything else here, and checks the `DbContext` we already own rather than opening a connection of its own. `Microsoft.Extensions.Diagnostics.HealthChecks` is dropped from the list because the ASP.NET Core shared framework already provides it — it was never a package we needed to add.
- `FluentAssertions` dropped, not replaced. Its version 8 licence change makes it a commercial dependency for some uses; plain xUnit assertions cost nothing here. Removing an approved package needs no approval, but it is recorded so a later reader does not add it back expecting it was always intended.

**fitforge-web**: `tailwindcss-animate`, `class-variance-authority`, `clsx`, `tailwind-merge`, `lucide-react`, `server-only` (all shadcn/ui prerequisites), and `next-themes` **only if** the hand-written theme script proves insufficient — preferred outcome is not adding it.

Explicitly **not** approved: MediatR, AutoMapper, FluentValidation, a repository/unit-of-work package, any state-management library, any component library other than shadcn/ui.

## 6. Phases

| # | Phase | Repository | Delivers |
|---|---|---|---|
| 1 | API layering and error contract | fitforge-api | The three projects, dependency direction, RFC 9457 handling, `/health/live`, WeatherForecast deleted |
| 2 | Persistence boundary and readiness | fitforge-api | `FitForgeDbContext`, validated options, `/health/ready` with the database check and its 503 path |
| 3 | Design tokens and app shell | fitforge-web | `globals.css` tokens, Inter, theme-before-paint, sidebar / bottom bar / header to VI-001…VI-013 |
| 4 | The BFF seam | fitforge-web | `api-client.ts`, `/api/health`, the health indicator in the shell, mapping tests |
| 5 | Binding the checks | governance + CI | `ritual-checks` required, out-of-Territory demonstration, `docs/onboarding.md` run instructions |
| 6 | Bound the readiness check | fitforge-api | Added 2026-09-10 — a defect phase 4's end-to-end observation found; see `tasks.md` |

Phase 3 is the UI phase: the Visual Compliance Loop (`docs/sdlc/review-process.md`) runs
against `screenshots/fitforge-prototype.html` until the deviation table is empty or
user-approved. It does not end when the code compiles.

## 7. Complexity Tracking

No constitution gate is being violated, so nothing is tracked here. The one judgement call
worth recording is §4.3's boundary definition: a stricter reading of FR-004 would forbid
`FitForgeDbContext` in `Api` handlers entirely and force a query-service layer. That was
weighed and rejected in §4.1 option 3 — the stricter rule buys purity the project cannot
spend and would be quietly broken by the third feature.
