# Implementation Plan: Identity and Member Profile

**Branch**: `002-identity-member-profile` | **Date**: 2026-09-10 | **Spec**: `specs/002-identity-member-profile/spec.md`
**Input**: Feature specification from `/specs/002-identity-member-profile/spec.md`
**Delivery Level**: Critical (`docs/sdlc/critical-delivery.md`)
**Gate Batching**: none
**Gate Certification**: user-run

Both lines above are the only legal values for a Critical feature.
`scripts/enforcement-pack.ps1` fails the branch on a declared batch or on `ci-held`
(critical-delivery item 4). Feature 001 was certified `ci-held`; that route is closed here,
and **every one of the ten phase gates below is run by a human, locally**. That is the
largest single cost this plan carries, and it is stated here rather than discovered at
phase 1.

## Summary

FitForge learns who is asking. Two business entities (`Member`, `Profile`), two supporting
tables (`Session`, `SignInAttempt`), an opaque server-side session the API can revoke, and a
cookie the BFF owns and the browser cannot read.

The centre of this plan is not any one endpoint — it is the **shape** of the API surface.
Every member-owned path is `/me`-shaped, with no parameter, header or body field naming
*which* member. Invariant 2 is then upheld by an absence rather than by a filter somebody
has to remember, and the reviewer's job becomes "is there a path that takes a member
identifier?" instead of "is every query filtered?".

Ten phases. The first six are API-only, the next three are web-only, and the last is
governance. No phase touches both code repositories (constitution III).

## Technical Context

**Language/Version**: C# / .NET 10 (SDK pinned in `global.json`); TypeScript 5 on Node 22
**Primary Dependencies**: as approved in 001 §5, plus exactly one addition — see §5 below
**Storage**: SQL Server, owned exclusively by `fitforge-api`. Two additive migrations
**Testing**: xUnit + `WebApplicationFactory` in `fitforge-api`; Vitest in `fitforge-web`
**Target Platform**: Linux/Windows server for the API; Node server for the web application
**Project Type**: Web application across two repositories (`fitforge-api`, `fitforge-web`)
**Performance Goals**: session resolution adds no more than one indexed lookup to an
authenticated request; the deliberate password stretch (D2) is on the two credential paths
only, never on session resolution
**Constraints**: both gate commands exit 0 at every phase boundary; `--warnaserror` stays on
**Scale/Scope**: 2 developers; 2 screens from the visual reference; 28 VI items; 18 FRs

## Constitution Check

*GATE: passed before Phase 0. Re-checked after the decisions in §4 were written.*

- [x] **Specification First (I)**: `spec.md` was approved 2026-09-10 before this plan; this
      plan and `tasks.md` land and are approved before any code phase.
- [x] **Source of Truth (II)**: no conflict. The prototype (rung 2) and this feature's
      captures (rung 1) agree; the three places this feature does *not* follow the reference
      are declared in `spec.md` ("Declared deviations"), not decided here.
      **One conflict was found and is reported, not resolved** — see §7.
- [x] **Repository Separation (III)**: phases 1–6 touch `fitforge-api` only, 7–9
      `fitforge-web` only, 10 governance only. No phase touches application code in both.
- [x] **Architecture Consistency (IV)**: ADR-001 (`specs/001-solution-scaffold/plan.md` §4)
      is the architecture and is followed without amendment. `FitForge.Domain` keeps its zero
      package references — the hashing dependency lands in `FitForge.Api` (D2). One package
      is added and approved in §5; nothing else may be.
- [x] **Domain Invariants (V)**: no rule in `modules/training/training-invariants.md` is
      violated. The item-by-item trace is in `data-model.md` ("Invariant trace") and is
      repeated in both reviews, per critical-delivery item 2.
- [x] **Security (VI)**: this feature *is* the authentication surface. Decisions D1–D7 are
      the substance of this box, not a checkmark on it. No secret enters source: the source
      salt (D5) and the connection string are names in configuration and values in the
      environment. No password, token, or source address is written to a log (D12).
- [x] **External Integration Governance (VII)**: `contracts/auth.md` and
      `contracts/member.md` are complete and approved **before** phase 1, and the BFF is
      implemented against them in phases 7–9, after the API side exists.
- [x] **Testing Requirements (VIII)**: every rule this feature enforces has a test that
      fails when the rule is removed — enumerated per phase in `tasks.md`, and specifically
      the four in D13 that exist to fail when a *later* feature breaks something.
- [x] **Human Review (IX)**: Ahmad reviews before merge; Anas owns. Per critical-delivery
      item 5 the reviewer is not the owner, and FitForge has two developers, so the
      solo-developer substitute (second-model review + 24-hour cooling-off) **does not
      apply**. See §6 — this is currently blocked on a kit change.
- [x] **Controlled Delivery (X)**: ten phases, one at a time, each independently revertible,
      each with a human-run gate. No batching, no `ci-held`.

**Phase sizing self-check.** Phases 3, 4 and 5 are all "the API's authenticated surface" and
are deliberately not one phase: reverting 5 leaves 4 correct and the gate green (sessions
issue and resolve, nothing member-owned is exposed); reverting 4 leaves 3 correct (the
session machinery exists and is tested, no credential path uses it yet). Phase 2 is small on
purpose — password hashing is the single most consequential decision in the feature and
earns its own gate and its own review pass rather than arriving inside a larger diff.
Phases 7 and 9 are separated because they are two different screens with two different
Visual Compliance Loops, and 8 is separated from both because route protection is provable
with no screen finished.

## Project Structure

### Documentation (this feature)

```text
specs/002-identity-member-profile/
├── spec.md                  # approved 2026-09-10
├── plan.md                  # this file
├── tasks.md
├── data-model.md
├── rollback.md              # Critical item 1 — filled BEFORE phase 1
├── contracts/
│   ├── auth.md
│   └── member.md
└── screenshots/             # 4 captures, sign-in and profile, light and dark
```

No `research.md`: every open question is decided in §4 rather than explored separately.

### Source Code

```text
fitforge-api/
├── src/FitForge.Domain/Members/          # NEW — Member, Profile, enums, email
│                                         #   normalization, password policy. Still
│                                         #   zero package references.
├── src/FitForge.Infrastructure/
│   ├── Persistence/Configurations/       # NEW — entity configurations
│   └── Migrations/                       # NEW — two additive migrations
└── src/FitForge.Api/
    ├── Features/Identity/                # NEW — register, sign-in, sign-out, session
    ├── Features/Me/                      # NEW — profile read, preferences, password, delete
    ├── Hosting/Authentication/           # NEW — bearer-token handler, CurrentMember
    └── Hosting/Retention/                # NEW — the daily purge service

fitforge-web/
└── src/
    ├── app/(auth)/sign-in/               # NEW — the unauthenticated screen, no shell
    ├── app/(app)/profile/                # NEW — the profile screen
    ├── app/api/bff/                      # NEW — the enumerated route handlers
    ├── lib/session.ts                    # NEW — cookie read/write, server-only
    └── lib/units.ts                      # EXISTS (001) — display conversion, extended
```

## 4. Decisions

Numbered so reviews and `tasks.md` can cite them.

### D1 — Password policy: length, and almost nothing else

Minimum **10** characters (FR-002's floor), maximum **256**. No composition rules — not
because they are harmless but because they measurably push people toward `Password1!` and
FR-002 forbids them outright.

Two rejections beyond length, both cheap and neither a composition rule:

- the password MUST NOT equal the email address, case-insensitively;
- input longer than 256 characters is rejected before hashing, so a 10 MB body cannot buy an
  attacker a 200 ms CPU stretch per request.

Enforced **server-side** in `FitForge.Domain` (a pure function, unit-tested with no host).
The browser may mirror it as a courtesy; invariant 7 means the browser's copy is never the
enforcement.

### D2 — Password hashing: PBKDF2-HMAC-SHA512, 210,000 iterations

`PasswordHasher<Member>` from `Microsoft.Extensions.Identity.Core`, configured with
`IterationCount = 210_000`. Its v3 format is HMAC-SHA512, a 128-bit random salt and a
256-bit subkey, written with a leading format byte.

- **210,000** is OWASP's current floor for PBKDF2-HMAC-SHA512. It is written as a named
  constant with that citation beside it, because the number is meant to be raised and a
  magic literal never gets raised.
- **The format byte is the reason for choosing this over hand-rolling.** It is what makes
  moving to Argon2id later a rehash-on-successful-sign-in rather than a forced reset for
  every member.
- **Rehash on verify** is implemented from day one: when the hasher reports
  `SuccessRehashNeeded`, the stored hash is rewritten inside the same request. Building the
  upgrade path now, while there is exactly one member to test it on, is the whole reason
  today's choice is reversible.

**Argon2id was not chosen, and it is the better algorithm.** The reason is dependency
governance, not cryptography: every Argon2 implementation available to us is third-party, in
the one place in this codebase where "we will review it later" is not an acceptable answer,
and FitForge has no one who can review a crypto library today. The decision is recorded as
reversible *because* of the rehash path above — this is a deferral with a mechanism, not a
preference.

**The BCL alternative** (`Rfc2898DeriveBytes.Pbkdf2` plus our own salt/format/versioning and
`CryptographicOperations.FixedTimeEquals`) was also weighed. It adds no package and is about
forty lines. It was rejected because those forty lines are format design and constant-time
comparison — small, easy to get subtly wrong, and worth exactly one first-party package to
not own.

The package reference lands in **`FitForge.Api`**, not `FitForge.Domain`. Hashing is
application logic under ADR-001 §4.2, and `FitForge.Domain` keeps its zero references — the
fact the ADR calls load-bearing, enforced by the csproj test 001 phase 8 corrected.

### D3 — Sessions: opaque, server-side, revocable, stored only as a hash

A 256-bit random token, base64url; the API stores `SHA-256(token)` and nothing else. Shape,
columns and the three approved deviations from `database-rules.md` are in `data-model.md`
("Session").

**A self-contained token (JWT) is excluded by the spec**, which requires FR-006 and FR-013 to
revoke server-side. A JWT plus a revocation list is a database lookup per request *and* a
token — strictly more machinery than the lookup alone. Recorded because "why not a JWT" is
the first question this design will be asked.

### D4 — The cookie the BFF owns

`__Host-fitforge_session`, `HttpOnly`, `Secure`, `SameSite=Lax`, `Path=/`, no `Domain`,
`Max-Age` tracking the session's expiry. Its value is the opaque token from D3.

- The **`__Host-` prefix** is the part worth naming: it makes the four properties above
  browser-enforced rather than server-promised, so a later handler cannot quietly widen the
  cookie's scope. It works on `http://localhost`, which browsers treat as a secure context,
  so development needs no exception.
- **`SameSite=Lax` plus an explicit `Origin` check** on every mutating BFF route is the CSRF
  answer. Lax already withholds the cookie from cross-site POSTs; the origin check is the
  belt to that suspenders, and it is three lines in one shared helper.
- **The cookie is not additionally sealed or encrypted.** HttpOnly already stops script
  reads, and a sealed blob is exactly as replayable as the token it wraps — sealing would
  add a key, a rotation story and a decrypt path in exchange for approximately nothing.

**Consequence, flagged rather than buried**: `FITFORGE_SESSION_SECRET` was added to
`fitforge-web/.env.example` by feature 001 in anticipation of a sealed cookie. This design
gives it no purpose, so phase 7 **removes it**. An environment variable that names a secret
nobody uses invites someone to make it load-bearing later without a decision.

### D5 — Throttling that cannot become an existence oracle

Two 15-minute fixed windows on failed attempts: 10 per normalized email, 30 per source.
Both answer **429** with `Retry-After`. Details in `contracts/auth.md` §6.

The load-bearing rule is that **the email bucket counts addresses that do not exist exactly
as it counts ones that do**. A throttle that only fires for real accounts turns
429-versus-401 into precisely the oracle D6 spends CPU to close — a real defect, and one
that would pass every test written from the happy path.

The source address is hashed with a configured salt before storage (`data-model.md`,
`SignInAttempt`). Fixed windows rather than sliding: a sliding window needs per-attempt
timestamps kept longer and buys accuracy nobody here can perceive.

### D6 — Indistinguishable failures, in body and in time

FR-004 requires unknown-email and wrong-password to be indistinguishable. Same status, same
body (`contracts/auth.md` §3) — and, when no member is found, the server verifies the
supplied password against a **fixed decoy hash** generated at startup with the same
parameters, so the two paths do the same work.

Without the decoy the timing difference is not subtle: one path is a 210,000-iteration
PBKDF2 and the other is an index miss. The test for this asserts that the decoy path
actually calls the hasher, rather than asserting on wall-clock timings — a timing assertion
in CI is a flaky test, not a security control.

**Registration is deliberately asymmetric**: `POST /auth/register` answers 409 for a taken
email. Recorded as a decision in `contracts/auth.md` §2 — a registration form that lies
about a taken address cannot complete, and the product has no cross-member surface for the
information to be worth anything.

### D7 — Member scoping by shape, not by filter

Every member-owned path is `/me`-shaped. There is no route parameter, header, or body field
naming a member. The authenticated member comes from the session and from nothing else.

This is a **structural** answer to invariant 2 and FR-008: the reviewer looks for a path that
takes a member identifier, and finding none is the proof. A filter-based design can only be
verified by reading every query and trusting that the next one is written the same way.

SC-002's test (phase 5) creates two members and asserts that every exposed identifier — B's
`PublicId` in a body, in a query string, in a header — leaves A reading A's data. It is
written to **fail** if the scoping is removed; a test that passes both ways is not evidence.

### D8 — Units are a rendering instruction

`Units` selects a display conversion in the browser. No write path converts, and no stored
value is rewritten when the preference changes (FR-011, invariant 4, VI-028).

Conversion lives in `fitforge-web/src/lib/units.ts` (already present from 001) and is used at
render time only. The guard is a test that asserts `PATCH /me/preferences` with a changed
`units` leaves every measurement column byte-identical — the direct statement of VI-028, and
the thing a future "helpful" refactor would break.

### D9 — Retention: a hosted service, and a guard that outlives this feature

A `BackgroundService` in `FitForge.Api` runs daily, permanently removing members whose
`DeletedAtUtc` is more than 30 days past, together with every row they own, and pruning
`SignInAttempt` rows older than their window. No package, no scheduler, no new deployment
surface.

**30 days is stated to the member in VI-027**, so the number lives in one named constant that
both the purge and the UI copy cite. If it changes, both change or neither does.

The guard that matters is not the purge — it is the test in D13 that fails when a later
feature adds a member-owned table and does not extend the purge. Invariant 10's second half
("unrecoverable after the stated window") is the half a soft delete cannot do, and it is the
half that silently rots as the schema grows.

### D10 — Time zones

Stored as an IANA identifier (FR-012), validated server-side with
`TimeZoneInfo.FindSystemTimeZoneById` — .NET 6+ resolves IANA identifiers on Windows and
Linux alike through ICU. An identifier the host cannot resolve is a 422 with a member-legible
message (`contracts/member.md` §2), never a silent fall back to UTC: a member whose streak is
computed in the wrong zone should be told, not quietly mis-served.

Nothing in 002 computes a day or week boundary — there is no training data yet. This decision
exists so that feature 007 inherits a validated zone rather than a free-text column.

### D11 — What the BFF may do, enumerated

Seven route handlers, listed in `contracts/member.md` §6. **No generic pass-through.** A
`/bff/proxy/[...path]` route would hand the browser the entire API surface behind a cookie
and make invariant 7 unenforceable by inspection.

The BFF maps and nothing else: it holds no domain rule, opens no database connection, and
never decides what a member may do. `api-client.ts` keeps its `import "server-only"` so a
client component importing it fails the build (ADR-001 §4.6).

### D12 — What is never logged

No password, no token, no `TokenHash`, no source address, and no internal `Id` reaches a log
sink, at any level, in any environment (invariant 8, constitution VI). Failed sign-ins log
the normalized email and the outcome; that is the audit value, and it is the boundary.

Problem documents carry no exception message or stack trace outside Development, per ADR-001
§4.4 — already true, restated because this is the first feature where the leak would matter.

### D13 — The four tests that exist to fail later

Ordinary coverage is enumerated in `tasks.md`. These four are different: each fails when a
*future* feature breaks something this feature established.

1. **Purge completeness** — enumerates `FitForgeDbContext` entity types carrying a member
   reference and asserts each is named in the retention purge (D9).
2. **No member parameter** — enumerates mapped endpoints under `/me` and asserts none
   declares a route parameter, query parameter, or body field naming a member (D7).
3. **Cross-member read** — SC-002's two-member test (D7).
4. **Units do not rewrite storage** — SC-002's sibling for invariant 4 (D8).

Test 2 is the one most likely to be deleted by someone who finds it annoying. That is the
argument for it, not against it, and this sentence is here so its next reader knows it was
put there on purpose.

## 5. Packages approved by this plan

001 §5's list carries forward unchanged. This plan adds **one** package and nothing else may
be added without amending this plan (constitution IV):

**fitforge-api**: `Microsoft.Extensions.Identity.Core` — for `PasswordHasher<TUser>` only
(D2), referenced by `FitForge.Api`. Not the ASP.NET Core Identity stack: no
`IdentityDbContext`, no `UserManager`, no `SignInManager`, no Identity UI, no
`AddIdentity(...)` registration. Those bring a schema and an opinion about authentication
that would displace this plan's design.

**fitforge-web**: **none**. The shadcn/ui primitives this feature needs (`input`, `label`,
`select`, `card`) are generated source under `src/components/ui/`, ours to edit — a
generator, not a runtime dependency (ADR-001 §4.6).

Explicitly **not** approved, restated because they are the plausible suggestions here:
NextAuth / Auth.js, `iron-session`, `jose` or any JWT library, any Argon2 or bcrypt package,
any rate-limiting package, `Microsoft.AspNetCore.Identity.EntityFrameworkCore`.

## 6. The independence requirement, and what it is blocked on

Critical item 5 requires a human reviewer who is not the owner. FitForge has two developers,
so Ahmad reviews and the solo substitute does not apply.

`scripts/enforcement-pack.ps1` cannot see that today. Until FitForge declares
`"developers": ["anas.m", "ahmad"]` in `kit-adoption.json` — which needs kit feature 013
merged and flowed down — the check reads this project as solo and demands a
`second-model-review.md` plus a 24-hour cooling-off that the actual arrangement makes
redundant.

**This does not block phases 1–10.** It blocks the merge, and it is named here so that it is
tracked as a dependency rather than discovered as a red CI run at the end. This feature is
kit 013's SC-001.

## 7. Conflict found, reported, not resolved

Constitution II's conflict rule: *if any two rungs conflict, stop and report — never silently
choose.*

`CLAUDE.md` ("Feature Structure") lists the spec directory's contents as **"exactly these
names"** — seven entries, which do not include `rollback.md`, `ai-code-review*.md`, or
`human-pr-review.md`. `docs/sdlc/branch-strategy.md` ("Spec Directory Contents") lists
fourteen entries and names all three explicitly as per-feature files from
`specs/_templates/`.

They cannot both be right, and the difference is not academic here: Critical item 1 requires
a filled rollback plan before phase 1, and only one of the two documents says where it goes.

**What this plan did**: filed it as `rollback.md`, per `branch-strategy.md` — the only
document that names the file at all, and the reading every feature in this project and in
the kit already follows (001 and every kit feature carry `ai-code-review*.md` in the spec
directory, which `CLAUDE.md`'s list also excludes).

**What is owed**: this is a defect in the kit, not in FitForge, and it is reported upstream
rather than patched locally. `CLAUDE.md`'s "exactly these names" is contradicted by the
kit's own practice on every feature it has ever shipped.

## 8. Phases

| # | Phase | Repository | Delivers |
|---|---|---|---|
| 1 | Members and profiles in the schema | fitforge-api | `Member`, `Profile`, configurations, `AddMemberAndProfile`. No endpoint |
| 2 | Password policy and hashing | fitforge-api | D1's pure policy in `Domain`; D2's hasher, parameters and rehash-on-verify in `Api`. No endpoint |
| 3 | Sessions | fitforge-api | `Session`, `SignInAttempt`, `AddSessionAndSignInAttempt`, issue/resolve/revoke, the bearer handler, `GET /auth/session` |
| 4 | Credentials | fitforge-api | `register`, `sign-in` (decoy hash, throttle), `sign-out` |
| 5 | The `/me` surface | fitforge-api | read, preferences, password change, delete; D13 tests 2, 3 and 4 |
| 6 | Retention | fitforge-api | The daily purge, the 30-day constant, D13 test 1 |
| 7 | Sign in and register | fitforge-web | The screen to VI-001…VI-016, the three auth BFF routes, the cookie, `FITFORGE_SESSION_SECRET` removed. **Visual Compliance Loop** |
| 8 | Route protection | fitforge-web | Redirect for unauthenticated routes, no shell on sign-in (VI-015), `GET /bff/me` |
| 9 | Profile | fitforge-web | The screen to VI-017…VI-028, the four member BFF routes, display conversion. **Visual Compliance Loop** |
| 10 | Audit evidence | governance | Critical item 3: gate commands and exit codes, scope-check verdicts, diff stats, both review checklists |

Phases 7 and 9 are the UI phases: the Visual Compliance Loop
(`docs/sdlc/review-process.md`) runs against this feature's `screenshots/` until the
deviation table is empty or holds only the three deviations `spec.md` declares. Neither
phase ends when the code compiles.

**Every phase's gate is run by a human** (critical-delivery item 4). No phase is claimed
done on an agent-run gate or a CI conclusion.

## 9. Complexity Tracking

No constitution gate is violated, so nothing is tracked as an exception. Three judgement
calls are recorded instead, because each is a place a reviewer could reasonably land
differently:

| Call | Chosen | The cost accepted |
|---|---|---|
| Password hash (D2) | PBKDF2 via a first-party package | Not Argon2id, which is stronger. Mitigated by rehash-on-verify, which is built in phase 2 rather than promised |
| Session store (D3) | A database row per session | A lookup per authenticated request. Bought: real revocation, which FR-006 and FR-013 require |
| Ten phases | Ten human-run gates | Slow. It is what Critical costs, and the alternative — fewer, larger phases — makes each revert wider exactly where reverts matter most |
