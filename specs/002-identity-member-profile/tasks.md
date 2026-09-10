# Tasks: Identity and Member Profile

**Feature**: `002-identity-member-profile` | **Plan**: `specs/002-identity-member-profile/plan.md`
**Owner**: anas.m | **Reviewer**: ahmad | **Delivery Level**: Critical

Territory entries are repo-prefixed and relative to the governance root, per
`docs/sdlc/repository-strategy.md` ("Territory across repositories"). The feature's own spec
directory is implicitly in territory and is never declared.

Each code phase commit lands on the matching `002-identity-member-profile` branch in its code
repository — the Cross-Repository Feature Rule is what lets `scripts/scope-check-repos.ps1`
find this declaration.

**Every gate below is run by a human, locally** (`docs/sdlc/critical-delivery.md` item 4).
No phase is done on an agent-run gate or a CI conclusion. The command, its exit code, the
`scope-check` verdict and `git diff --stat` are recorded in the phase's Gate block as
evidence (item 3).

## Format: `[ID] [P?] [Story] Description`

`[P]` marks tasks that may run in parallel (different files, no ordering dependency).
Plan decisions are cited as `D1`…`D13`; visual items as `VI-nnn`; requirements as `FR-nnn`.

---

## Phase 1: Members and profiles in the schema (US1)

**Territory**:

- `fitforge-api/src/FitForge.Domain/**`
- `fitforge-api/src/FitForge.Infrastructure/**`
- `fitforge-api/tests/**`

### Implementation

- [x] T001 Add `src/FitForge.Domain/Members/Member.cs` — the entity of `data-model.md`, with `Id`, `PublicId`, the audit fields and the soft-delete fields. No attribute from any package: `FitForge.Domain` still references nothing (ADR-001 §4.2).
- [x] T002 [P] Add `src/FitForge.Domain/Members/Profile.cs` — `BirthYear`, `Sex`, `HeightCm` as `decimal?`, plus audit and soft-delete fields.
- [x] T003 [P] Add `src/FitForge.Domain/Members/UnitPreference.cs`, `Goal.cs`, `ExperienceLevel.cs`, `Sex.cs` — enums whose member order matches VI-019 to VI-021, since the select order is a visual requirement and the enum is where it will be read from.
- [x] T004 Add `src/FitForge.Domain/Members/EmailAddress.cs` — normalization as a pure function: trim, then `ToUpperInvariant`. One place, used by the entity and by every query (`data-model.md`, `Member`).
- [x] T005 Add `src/FitForge.Infrastructure/Persistence/Configurations/MemberConfiguration.cs` — column types exactly as `data-model.md` states, `UQ_Member_NormalizedEmail`, `UQ_Member_PublicId`, and a check constraint bounding each enum column to its defined range (`database-rules.md`, "Constraints Mirror Invariants").
- [x] T006 [P] Add `ProfileConfiguration.cs` — `FK_Profile_Member` with **restrict**, `UQ_Profile_MemberId`, `CK_Profile_BirthYear`, `HeightCm` as `DECIMAL(5,2)`.
- [x] T007 Add the two `DbSet` properties to `FitForgeDbContext` and a global query filter excluding soft-deleted rows (`database-rules.md`).
- [x] T008 Confirm the target database is dedicated to FitForge **before generating the first migration** — `database-rules.md`, "Setup". This is the first migration in the project's life and the check exists for exactly this moment.
- [x] T009 Generate migration `AddMemberAndProfile`. Read the generated SQL before committing it; a migration nobody read is a migration nobody can roll back.

### Tests

- [x] T010 [P] `FitForge.Domain.Tests` — email normalization: mixed case, leading and trailing whitespace, and the two forms colliding.
- [x] T011 `FitForge.Api.Tests` — the migration's `UpOperations` create exactly `Member` and `Profile`, and its `DownOperations` drop exactly those two, child first (`rollback.md` asserts the down-path is safe; the assertion should be executed, not believed). **Amendment approved by**: anas.m, 2026-09-10 — was "applies to an empty database"; no database is reachable from the authoring host or from CI, and the operations settle the same question without one (A1).
- [x] T012 `FitForge.Domain.Tests` — the ADR guard still passes: `FitForge.Domain.csproj` declares no `PackageReference` and no `ProjectReference`. It exists from 001; this phase is the first real chance to break it.

**T008 — the dedicated-database confirmation, recorded.** The configured target is
`Server=localhost;Database=FitForgeDev;Trusted_Connection=True` (from user-secrets; the
value is never in source). The name is FitForge's own, not one inherited from another
project's local setup — which is the trap `database-rules.md` "Setup" describes. **What
could not be confirmed**: the SQL Server instance was not reachable from the authoring
host, so the database's *contents* were not inspected. The name check passed; the
emptiness check is owed to whoever first applies this migration, and it belongs in the
gate run rather than in this record.

**Gate (human-run) — critical-delivery item 3, audit evidence**

| | |
|---|---|
| Command | `dotnet build --warnaserror && dotnet test` in `fitforge-api` |
| **Exit code** | **0**, run by anas.m, 2026-09-10 |
| Commit gated | `3279d50` |
| `scope-check` | `PASS phase 1 commit 9a5bf35 (1 file(s))` — the governance side |
| `scope-check-repos` | `PASS phase 1 commit 3279d50 (15 file(s))` — `fitforge-api` |
| `git diff --stat` | 15 files changed, 1066 insertions(+), 2 deletions(-) |

**The scope check failed first, and that is part of the record.** The initial phase 1
commit `f4cd9e1` was rejected by `scripts/scope-check-repos.ps1` for touching
`fitforge-api/src/FitForge.Api/FitForge.Api.csproj`, outside the declared Territory. It
was reverted rather than licensed by a widened Territory — see A3 below. `3279d50` is the
re-commit. A Critical feature's audit trail should show the check working, not only the
green line it eventually produced.

**Second gate run — the commit that completes the phase**

T011 landed under amendment A1 *after* the run above, so the state that was gated and the
final state of the phase are not the same commit. Recorded as two runs rather than folded
into one: saying otherwise would make this table a claim instead of a record.

| | |
|---|---|
| Command | `dotnet build --warnaserror && dotnet test` in `fitforge-api` |
| **Exit code** | **0**, run by anas.m, 2026-09-10 |
| Commit gated | `7922e31` |
| `scope-check-repos` | `PASS phase 1 commit 7922e31 (1 file)` |
| `git diff --stat` | 1 file changed, 143 insertions(+) |

**Phase 1 is done**: T001–T012 complete, both gate runs exit 0, both scope checks PASS.

T011's six assertions were **mutation-checked**, not assumed: flipping the expected
`OnDelete` to `Cascade` and the expected drop order to Member-first fails two of the six.
A test that passes both ways is not evidence, and this phase's own `spec.md` says so
about SC-002 — the same standard applies to the tests that reach it.

---

## Phase 2: Password policy and hashing (US1)

**Territory**:

- `fitforge-api/src/FitForge.Domain/**`
- `fitforge-api/src/FitForge.Api/**`
- `fitforge-api/tests/**`

### Implementation

- [x] T013 Add `src/FitForge.Domain/Members/PasswordPolicy.cs` — D1 as a pure function returning the violated rule or nothing: minimum 10, maximum 256, and not equal to the email case-insensitively. No composition rules (FR-002 forbids them).
- [x] T014 ~~Add `Microsoft.Extensions.Identity.Core` to **`FitForge.Api` only**~~ — **not added; none was needed.** The shared framework provides `PasswordHasher<TUser>` on .NET 10, and referencing the package raises NU1510, which `TreatWarningsAsErrors` makes a build failure. Feature 002 adds no package. Recorded in `plan.md` §5; a removal needs no approval. `FitForge.Domain` confirmed still at zero package references.
- [x] T015 Add `src/FitForge.Api/Features/Identity/PasswordHashing.cs` — `PasswordHasher<Member>` configured with `IterationCount = 210_000` as a named constant carrying its OWASP citation (D2). A magic literal never gets raised.
- [x] T016 Implement **rehash on verify**: when the hasher reports `SuccessRehashNeeded`, rewrite the stored hash inside the same request. This is what makes D2 reversible, and it is built now while there is one member to test it on.
- [x] T017 Register the hasher in DI, and generate the **fixed decoy hash** at startup with the same parameters (D6). It is a field, not a per-request computation.

### Tests

- [x] T018 [P] `FitForge.Domain.Tests` — the policy: 9 characters rejected, 10 accepted, 257 rejected, password equal to the email rejected in either case.
- [x] T019 `FitForge.Api.Tests` — a hash verifies against its own password and fails against a different one; two hashes of the same password differ (the salt is real).
- [x] T020 `FitForge.Api.Tests` — rehash-on-verify: a hash produced at a lower iteration count verifies **and** is rewritten. The test asserts the stored value changed, not that a method was called.

**Gate (human-run) — critical-delivery item 3, audit evidence**

| | |
|---|---|
| Command | `dotnet build --warnaserror && dotnet test` in `fitforge-api` |
| **Exit code** | **0**, run by anas.m, 2026-09-10 |
| Commit gated | `4b56aa7` |
| `scope-check-repos` | `PASS phase 2 commit 4b56aa7 (6 file(s))` |
| `git diff --stat` | 6 files changed, 628 insertions(+) |

**Mutation-checked, not assumed.** The two assertions this phase rests on were each broken
on purpose to confirm they fail:

| Mutation | Result |
|---|---|
| `Verify` stops rewriting `PasswordHash` on `SuccessRehashNeeded` | T020 fails |
| `VerifyDecoy` returns before verifying | the decoy test fails |

Both reverted; 14 of 14 pass. A test that passes both ways is not evidence, and for the
two mechanisms that make D2 reversible and FR-004 true that standard is not optional.

---

## Phase 3: Sessions (US2)

**Territory**:

- `fitforge-api/src/FitForge.Domain/**`
- `fitforge-api/src/FitForge.Infrastructure/**`
- `fitforge-api/src/FitForge.Api/**`
- `fitforge-api/tests/**`

### Implementation

- [x] T021 Add `src/FitForge.Domain/Members/Session.cs` and `SignInAttempt.cs` per `data-model.md`, including the three approved deviations from `database-rules.md` (D3) — each one commented with the plan decision that approved it, so the next reader finds the reason and not just the absence.
- [x] T022 Add `SessionConfiguration.cs` and `SignInAttemptConfiguration.cs` — `UQ_Session_TokenHash`, `IX_Session_MemberId`, `IX_SignInAttempt_Email_At`, restrict FKs.
- [x] T023 Generate migration `AddSessionAndSignInAttempt`. One migration per phase (`database-rules.md`).
- [x] T024 Add `src/FitForge.Api/Features/Identity/SessionService.cs` — issue (256 random bits from `RandomNumberGenerator`, store `SHA-256` only), resolve by hash, revoke one, revoke all-but-one, revoke all. The token is returned to the caller exactly once and never read back from storage (D3).
- [x] T025 Add `src/FitForge.Api/Hosting/Authentication/BearerSessionHandler.cs` — turns `Authorization: Bearer <token>` into the current member, or 401. Expired, revoked, unknown and "belongs to a soft-deleted member" all produce the same 401 (`contracts/auth.md` §5).
- [x] T026 Add `CurrentMember` as the only way a handler learns who is asking. There is no other accessor, so D7 has one place to be right.
- [x] T027 Implement sliding expiry — extend to now + 14 days on resolve, at most once per hour, so the hot path is not a write per request.
- [x] T028 Map `GET /api/v1/auth/session` per `contracts/auth.md` §5.

### Tests

- [~] T029 → **moved to phase 11 as T105** (A4, approved anas.m 2026-09-10): it cannot precede the database wiring it needs.
- [~] T030 → **moved to phase 11 as T106**.
- [~] T031 → **moved to phase 11 as T107**.
- [x] T031b Partial coverage that needs no database: every shape of missing or malformed credential is refused identically, and an unauthenticated request never reaches persistence. Named as partial in the file rather than left to look like coverage it is not.

**Phase 3 is complete as scoped**, with T029–T031 moved to phase 11 under A4. It gates on
the implementation plus T031b; the tests that prove its central mechanism arrive one phase
later, a cost argued in A4 rather than glossed.

**Gate (human-run) — critical-delivery item 3, audit evidence**

| | |
|---|---|
| Command | `dotnet build --warnaserror && dotnet test` in `fitforge-api` |
| **Exit code** | **0**, run by anas.m, 2026-09-10 |
| Commit gated | `c4c817e` |
| `scope-check-repos` | `PASS phase 3 commit c4c817e (16 file(s))` |
| `git diff --stat` | 16 files changed |

---

## Phase 4: Credentials (US1)

**Territory**:

- `fitforge-api/src/FitForge.Api/**`
- `fitforge-api/appsettings.json`
- `fitforge-api/tests/**`

### Implementation

- [x] T032 Map `POST /api/v1/auth/register` per `contracts/auth.md` §2 — validate with D1, normalize with T004, create `Member` **and** `Profile` in one transaction (`data-model.md`: no read path handles a missing profile), issue a session, return 201.
- [x] T033 Handle the duplicate email as 409, and rely on `UQ_Member_NormalizedEmail` to be the actual arbiter — two concurrent registrations of the same address must produce one member and one 409, not two members.
- [x] T034 Map `POST /api/v1/auth/sign-in` per `contracts/auth.md` §3 — one message and one status for unknown email, wrong password and soft-deleted member.
- [x] T035 Implement the **decoy hash path** (D6): when no member is found, verify the supplied password against the startup decoy so both paths do the same work.
- [x] T036 Implement the throttle (D5, `contracts/auth.md` §6) — two 15-minute fixed windows, 10 per normalized email and 30 per source, both answering 429 with `Retry-After`. **Attempts against addresses that do not exist are counted identically**; this is the requirement, not an implementation detail.
- [x] T037 Add the source-address salt as a configuration **name** with an empty value in `appsettings.json`, documented in the repository README. No secret enters source (constitution VI).
- [x] T038 Map `POST /api/v1/auth/sign-out` — revoke server-side, 204, idempotent (FR-006).
- [x] T039 Audit every log statement added in this phase against D12: no password, no token, no hash, no source address, no internal `Id`. **Result: this phase adds no log statement at all.** Recorded as a finding rather than a tick — "nothing to audit" is the honest outcome, and it also means the audit trail a failed sign-in ought to leave (D12 allows the normalized email and the outcome) does not exist yet. Noted for phase 10 rather than added here: logging is not in this phase's task list, and adding it unasked is the scope creep the ritual exists to prevent.

### Tests

- [x] T040 `FitForge.Api.Tests` — register, then sign in, then resolve the session. The whole of US1 in one test.
- [x] T041 [P] `FitForge.Api.Tests` — unknown email and wrong password return byte-identical bodies and the same status (FR-004).
- [x] T042 `FitForge.Api.Tests` — the decoy path **calls the hasher**. Asserted through a counting hasher, not through wall-clock timing: a timing assertion in CI is a flaky test, not a security control (D6).
- [x] T043 [P] `FitForge.Api.Tests` — registering an email differing only in case, or by surrounding whitespace, is refused as a duplicate (FR-001, spec Edge Cases).
- [x] T044 `FitForge.Api.Tests` — the throttle fires on the 11th failure for an email **that does not exist**, and answers 429 exactly as it does for one that does (D5). This is the oracle test; without it the feature's headline defence is untested.
- [x] T045 `FitForge.Api.Tests` — a successful sign-in clears the email bucket and leaves the source bucket intact.
- [x] T046 `FitForge.Api.Tests` — sign-out revokes server-side: the token fails on the next resolve, and a second sign-out with the same token is still 204.

### What the tests caught that review would not have

| Finding | Where it was |
|---|---|
| Validation returned **400**, not the 422 `contracts/auth.md` §2 fixes | **the code.** `Results.ValidationProblem` defaults to 400, which is quietly plausible; the BFF branches on the status |
| Two failure bodies are not byte-identical | **the test.** `traceId` is per-request. Excluding that one named field is not a weakening — a correlation id random in both cases carries no information about which occurred |
| A second validated option changed the startup exception's shape | **the test.** Two failing options raise an `AggregateException`, not a bare `OptionsValidationException`. The host names **both** settings, which is better than before, so the test now asserts on flattened text and a new test pins the both-at-once behaviour |

**Mutation-checked.** Removing `RecordFailureAsync` from the no-such-member path — so the
throttle counts only real accounts — fails 2 of 15. That mutation *is* the existence
oracle: 429-versus-401 would become the answer to "does this address have a member?", and
the decoy hash's 210,000 iterations would be protecting a door with a window next to it.

**Gate (human-run) — critical-delivery item 3, audit evidence**

| | |
|---|---|
| Command | `dotnet build --warnaserror && dotnet test` in `fitforge-api` |
| **Exit code** | **0**, run by anas.m, 2026-09-10 |
| Commit gated | `479d43f` |
| `scope-check-repos` | `PASS phase 4 commit 479d43f (9 file(s))` |
| `git diff --stat` | 9 files changed, 937 insertions(+), 5 deletions(-) |

---

## Phase 5: The `/me` surface (US3, US4)

**Territory**:

- `fitforge-api/src/FitForge.Api/**`
- `fitforge-api/tests/**`

### Implementation

- [x] T047 Map `GET /api/v1/me` per `contracts/member.md` §1 — member plus profile, every profile field nullable, no internal `Id` in the payload (invariant 8).
- [x] T048 Map `PATCH /api/v1/me/preferences` §2 — units, goal, experience, time zone; absent means unchanged; `null` is not accepted for any of the four.
- [x] T049 Validate the time zone with `TimeZoneInfo.FindSystemTimeZoneById` and return 422 with a member-legible message when it does not resolve (D10). Never a silent fall back to UTC.
- [x] T050 Map `POST /api/v1/me/password` §3 — verify the current password, apply D1 to the new one, revoke **every other** session, keep the presented one (FR-013).
- [x] T051 Map `DELETE /api/v1/me` §4 — re-authenticate with the password, then in one transaction soft-delete the member and everything they own and revoke every session including the presented one (FR-014).
- [x] T052 Confirm by inspection that no path under `/me` declares a route parameter, query parameter, or body field naming a member (D7). T054 turns this from a habit into a check.

### Tests

- [x] T053 **[D13-3]** `FitForge.Api.Tests` — SC-002: two members; A's session with B's `PublicId` supplied in a body, a query string and a header returns A's data every time. Then remove the scoping and confirm the test **fails** — a test that passes both ways is not evidence.
- [x] T054 **[D13-2]** `FitForge.Api.Tests` — enumerate mapped endpoints under `/me` and assert none declares a member-naming parameter. This is the test most likely to be deleted by someone who finds it annoying; that is the argument for it.
- [x] T055 **[D13-4]** `FitForge.Api.Tests` — changing `units` leaves every measurement column byte-identical (FR-011, invariant 4, VI-028).
- [x] T056 [P] `FitForge.Api.Tests` — password change: the old password stops working, the new one works, other sessions are dead and the presented one survives.
- [x] T057 [P] `FitForge.Api.Tests` — a wrong current password refuses the change and leaves the existing password working.
- [x] T058 `FitForge.Api.Tests` — after deletion, sign-in returns the same 401 as a wrong password: a deleted account is not discoverable (FR-014).
- [x] T059 [P] `FitForge.Api.Tests` — an unresolvable time-zone identifier is a 422, not a silent UTC (spec Edge Cases).
- [x] T060 `FitForge.Api.Tests` — two concurrent password changes: one wins, the other is refused, and the account is never left with neither password working.

### SC-002 is met, and it fails when the scoping is removed

The spec required that: *"The test exists and fails when the scoping is removed."* Proven,
not asserted. `GET /me` was mutated to honour a caller-supplied `memberId` — the exact
defect invariant 2 calls "of the highest severity" — and the cross-member test failed.
Reverted; 4 of 4 pass.

**But only 1 of the 4 failed, and that is worth knowing.** T054, the structural test, did
**not** catch it: the mutation read the identifier from `HttpContext.Request.Query`
instead of declaring a parameter, and T054 sees declared parameters only. So:

| Test | Proves | Blind to |
|---|---|---|
| T054 (structural) | no endpoint under `/me` *declares* a way to name a member | a handler reaching into `HttpContext` directly |
| T053 (behavioural) | member A never receives member B's data | nothing here — it is the backstop |

Neither is sufficient alone, and the gap between them is now written down rather than
assumed away. A reviewer reading only T054 would over-trust it.

**D13-4 also mutation-checked**: making a units change convert the stored height — the
"helpful" refactor invariant 4 exists to forbid — fails T055.

### Three things the tests caught during this phase

| Finding | Where |
|---|---|
| `DELETE /api/v1/me` would not start: Minimal APIs do not *infer* a body for DELETE | **the code.** Bound explicitly rather than changing an approved contract; a password in a query string was never an option |
| T054 flagged `CurrentMember.Member` | **the test.** Correct by its own rule. Excluded by *type*, not by parameter name — the weak fix would have left the check blind to a dangerous type with a different name |
| T054 then flagged `FitForgeDbContext.Members` | **the test, again.** The rule is now "can the container supply it?", which is exact and stays exact as services are added |

**Gate (human-run) — critical-delivery item 3, audit evidence**

| | |
|---|---|
| Command | `dotnet build --warnaserror && dotnet test` in `fitforge-api` |
| **Exit code** | **0**, run by anas.m, 2026-09-10 |
| Commit gated | `43e30e6` |
| `scope-check-repos` | `PASS phase 5 commit 43e30e6 (4 file(s))` |
| `git diff --stat` | 4 files changed, 992 insertions(+) |

---

## Phase 6: Retention (US4)

**Territory**:

- `fitforge-api/src/FitForge.Api/**`
- `fitforge-api/tests/**`

### Implementation

- [x] T061 Add `src/FitForge.Api/Hosting/Retention/RetentionService.cs` — a `BackgroundService` running daily (D9). No package, no scheduler.
- [x] T062 Add the retention window as one named constant, **30 days**, cited by both the purge and the UI copy that phase 9 renders (VI-027). If it changes, both change or neither does.
- [x] T063 Permanently remove members whose `DeletedAtUtc` is more than the window past, together with every row they own, in one transaction per member.
- [x] T064 Prune `SignInAttempt` rows older than their 15-minute window — the table is a counter, not a log, and keeping it is keeping personal data past its usefulness (invariant 10).
- [x] T065 Log what was removed as counts only: never an email, never an address, never an internal `Id` (D12).

### Tests

- [x] T066 **[D13-1]** `FitForge.Api.Tests` — enumerate every `FitForgeDbContext` entity type carrying a member reference and assert each is named in the purge. A later feature adding a member-owned table without extending the purge fails the gate rather than silently orphaning personal data (D9).
- [x] T067 [P] `FitForge.Api.Tests` — a member soft-deleted 31 days ago is removed; one soft-deleted 29 days ago is not; one not deleted at all is not.
- [x] T068 `FitForge.Api.Tests` — the purge removes the member's `Profile` and `Session` rows too, leaving no orphan.

### D13-1 mutation-checked in both directions

The guard is the point of this phase, so it was attacked rather than admired:

| Mutation | Result |
|---|---|
| drop `Profile` from `RetentionPolicy.Purged` — a member-owned entity nobody purges | fails |
| add `"WorkoutSession"` to the set — a name for an entity that does not exist | fails |

The second direction matters as much as the first. Without it, a stale name left behind
after a rename would let the completeness test pass while comparing against a schema
nobody has any more — green for the wrong reason, which is the failure mode this whole
feature keeps finding.

### Two decisions a reviewer should weigh

**A first pass runs five minutes after startup, not a day later.** An instance restarted
daily would otherwise never purge anything: the timer would reset before it ever fired,
and invariant 10's window would quietly never close. The bug would be invisible — nothing
errors, nothing logs, data simply is not erased.

**One transaction per member, not one for the batch.** A failure part way through leaves
earlier members fully removed and later ones untouched, never a member half-removed with
rows referencing nothing. The foreign keys are RESTRICT, so children are deleted first —
that ordering is the schema's protection working, not an obstacle routed around.

**Logs carry counts only** (D12). "Which member was erased" is precisely the fact erasure
exists to destroy, so writing it to a log would undo the work in the same breath.

**Gate (human-run) — critical-delivery item 3, audit evidence**

| | |
|---|---|
| Command | `dotnet build --warnaserror && dotnet test` in `fitforge-api` |
| **Exit code** | **0**, run by anas.m, 2026-09-10 |
| Commit gated | `2233d8e` |
| `scope-check-repos` | `PASS phase 6 commit 2233d8e (5 file(s))` |
| `git diff --stat` | 5 files changed, 468 insertions(+) |

**The API side of feature 002 is complete after this phase.** Phases 7–9 are `fitforge-web`.

---

## Phase 7: Sign in and register (US1, US2) — UI phase

**Territory**:

- `fitforge-web/src/**`
- `fitforge-web/.env.example`
- `fitforge-web/components.json`
- `fitforge-web/package.json`
- `fitforge-web/package-lock.json`

### Implementation

- [x] T069 Generate the shadcn/ui primitives this feature needs — `input`, `label`, `select`, `card` — into `src/components/ui/`. Generated source, not a runtime dependency (plan §5).
- [x] T070 Add `src/lib/session.ts` — read, write and clear `__Host-fitforge_session` with `HttpOnly`, `Secure`, `SameSite=Lax`, `Path=/`, no `Domain` (D4). `import "server-only"`.
- [x] T071 Add the shared `Origin`-check helper and apply it to every mutating BFF route (D4). Three lines, one place.
- [x] T072 [P] Add `src/app/api/bff/auth/sign-in/route.ts`, `register/route.ts`, `sign-out/route.ts` per `contracts/member.md` §6 — map, set or clear the cookie, and nothing else. **No generic pass-through route** (D11).
- [x] T073 Map upstream unavailability to a service failure, never a credential failure (FR-017, `contracts/auth.md` §7). A member told their password is wrong when the server is down will change a password that was fine.
- [x] T074 Build the sign-in screen at `src/app/(auth)/sign-in/page.tsx` to VI-001 through VI-016 — two-column grid at ≥1024px, left panel not rendered below it (VI-001), segmented control with Sign in selected (VI-007), field order email then password (VI-008), error above the button (VI-012), full-width 40px submit (VI-013).
- [x] T075 Render the three declared deviations exactly as `spec.md` declares them: no "Forgot?" link (so VI-010's row is the label alone), and the register half of the segmented control switches the same card rather than routing away.
- [x] T076 **Remove `FITFORGE_SESSION_SECRET` from `.env.example`** (D4). This design gives it no purpose, and a named secret nobody uses invites someone to make it load-bearing later without a decision.

### Tests and the loop

- [x] T077 [P] Vitest — the BFF sign-in route maps 401 to a credential error and an unreachable API to a service error. Every row of the mapping table, as 001 did for health.
- [x] T078 Vitest — the cookie is set with all five attributes, and its value is not readable from a non-`HttpOnly` path.
- [x] T079 **Visual Compliance Loop** (`docs/sdlc/review-process.md`) against `screenshots/01-signin-desktop-{light,dark}.jpg`, in both themes, until the deviation table is empty or holds only the three `spec.md` declares.
- [x] T080 Verify the responsive rules live at <1024px (VI-001, VI-017). No capture exists below 1024px — `spec.md` says so — so these are checked in a resized browser, not against an image.
- [x] T081 **SC-003 by inspection**: after sign-in, `localStorage`, `sessionStorage` and every script-readable cookie hold nothing that authenticates, and the browser issues no request to the API origin. Recorded in this file with what was inspected.

### Visual Compliance Loop — result

Run against `screenshots/01-signin-desktop-{light,dark}.jpg` and the prototype, measuring
the rendered page rather than eyeballing it. Every VI item measured, in both themes and
at five widths.

| VI | Required | Measured |
|---|---|---|
| VI-001 | grid `1fr 380px`, gap 24px at ≥1024px | `844px 380px`, gap 24px |
| VI-002 | panel padding 40px, rounded, distributed top/middle/bottom | 40px, radius 10.4px, `space-between` |
| VI-003 | mark 28×28 stroked in primary; wordmark 20px semibold; gap 8px | 28×28, `rgb(245,96,10)` = `--primary`; 20px/600; 8px |
| VI-004 | headline 24px semibold snug, max 24rem; body 14px muted, 12px below | 24px/600, line-height 33px, max-width 384px; 14px `rgb(113,113,122)`, 12px |
| VI-005 | 12px muted | 12px muted |
| VI-006 | card rounded, 1px border, padding 24px | radius 10.4px, padding 24px, border 1px |
| VI-007 | segmented on `--muted`, 4px inset, two equal buttons, Sign in selected | `rgb(244,244,245)`, 4px, `159.333px 159.333px`, Sign in default |
| VI-008 | Email then Password, labels 14px medium | order confirmed, 14px/500 |
| VI-009 / VI-011 | inputs 40px, 12px padding, 14px text | both 40px, 12px, 14px |
| VI-012 | error under password, above the button, 12px destructive, hidden when none | hidden with no error; renders in that position with one |
| VI-013 | submit full width, 40px | 331px (the card's inner width), 40px |
| VI-014 | footer 16px below, centred, 12px muted | 16px, centre, 12px |
| VI-015 | no app shell | no `<header>`, no `<nav>` — structural, see below |
| VI-016 | themes differ only in token values | geometry byte-identical across both; colours differ |

**Deviation found and fixed — the loop earning its keep.** VI-001 says that below 1024px
it is *"the card alone, centered"*. The first version was centred and **wrong**: the
single-column grid stretched the card to the full viewport — 868px at 900px wide, 991px
at 1023px. Centred, yes; a sign-in card, no. Capped at 380px below `lg`, then re-measured:

| Width | Panel | Card | Centred | Sideways scroll |
|---|---|---|---|---|
| 1280 | rendered | 380px | n/a (two columns) | no |
| 1023 | **not rendered** | 380px | yes | no |
| 900 | not rendered | 380px | yes | no |
| 768 | not rendered | 380px | yes | no |
| 390 | not rendered | 358px (page padding) | yes | no |

**T080's method, recorded because it is not the obvious one.** The window would not
resize below 1280px — the same limit `spec.md` records for the captures. Media queries
were therefore evaluated in **iframes** of the target width, which get their own viewport
for that purpose. Measured, not inferred from the class names.

**VI-015 is structural, not conditional.** The app shell moved out of the root layout
into `src/app/(app)/layout.tsx`, and sign-in lives in `(auth)`. A conditional inside the
shell is one `if` away from leaking navigation onto a screen a visitor with no account
should never see; a route group cannot do that, and every screen added to the group
inherits the right answer.

**Declared deviations, rendered as declared** (`spec.md`): no "Forgot?" link — so VI-010's
row holds the label alone — and Register switches the card in place rather than routing
away.

One measurement worth not over-reading: computed border width reads `0.666667px` rather
than `1px`. That is the display's device-pixel ratio of 1.5 snapping a 1px border to one
device pixel, not a CSS deviation — the declared value is `1px`.

### SC-003 — verified by inspection, both halves

Against a real signed-in session: the API running on LocalDB, an account registered, and
sign-in performed through `/api/bff/auth/sign-in`, the exact call `SignInCard.submit()`
makes.

| Checked | Result |
|---|---|
| `localStorage` | empty |
| `sessionStorage` | empty |
| `document.cookie` | empty — **zero** script-readable cookies |
| IndexedDB | no databases |
| any occurrence of a token, the password, or "bearer" in script-reachable storage | none |
| requests the browser made | `/api/bff/auth/sign-in` only — **no request to the API origin** (`127.0.0.1:5099`) |

**The half that is easy to fake, and how it was actually settled.** An empty
`document.cookie` proves nothing on its own — it reads the same whether the cookie is
HttpOnly or was never set. So: `POST /api/bff/auth/sign-out` was called, and the database
then showed **one session revoked** out of three. The BFF can only have revoked it by
sending a token it read from a cookie the script could not see. Both halves of SC-003 hold.

*Caveat, stated rather than glossed:* the sign-in was driven by `fetch` from the page, not
by typing into the form — synthetic keystrokes would not reach the focused field in this
environment. The call is identical in path, headers and body to the component's, and the
component's own mapping is covered by T077, but a human running the form by hand is still
owed at review.

**Gate (human-run)**: `npm run lint && npm run build && npm run typecheck && npm test` in
`fitforge-web`.

| | |
|---|---|
| **Exit code** | **0**, run by anas.m, 2026-09-10 |
| Commit gated | `ea76672` |
| `scope-check-repos` | `PASS phase 7 commit ea76672 (21 file(s))` |
| `git diff --stat` | 20 files changed, 957 insertions(+), 18 deletions(-) |

---

## Phase 8: Route protection (US2)

**Territory**:

- `fitforge-web/src/**`

### Implementation

- [x] T082 Redirect an unauthenticated visitor from every authenticated route to sign-in, rendering **no** member data on the way (FR-007).
- [x] T083 Render the sign-in route with **no app shell** — no sidebar, no bottom bar, no header (VI-015). This is a route-group decision, not a conditional inside the shell.
- [x] T084 Add `GET /api/bff/me` and have the shell read the member from it server-side.
- [x] T085 Ensure the back button after sign-out does not restore an authenticated view — the response carries no-store, and the shell re-reads the session on every navigation (US1 scenario 5).

### Tests

- [x] T086 [P] Vitest — an unauthenticated request to an authenticated route redirects and renders no member data.
- [x] T087 Vitest — the sign-in route renders no shell element.

### One decision, in one place

The gate is the `(app)` layout, and **nothing else decides who is signed in**. The
tempting second mechanism — a cookie-presence check in the proxy, for a cheap early
redirect — was rejected: a cookie proves someone *once* had a session, not that they
still do. Expired, revoked and belonging-to-a-deleted-member are indistinguishable from
the cookie and identical to the layout, so a proxy check would be a second opinion that
disagrees with the authoritative one on exactly the cases that matter.

Doing it in the **layout** rather than in each page is what makes FR-007's second half
true — "without rendering member data". The redirect is thrown before any child renders,
so there is no window in which a page runs without a member and improvises. T086 asserts
that directly: the layout returns nothing, not merely a redirect status.

### T085 — the header lands in production, and not in dev

Measured on both, because the difference would mislead a reviewer:

| Response | dev | production |
|---|---|---|
| `/` (authenticated) | `no-cache, must-revalidate` — **Next's own header wins** | `no-store, must-revalidate` — the proxy's |
| `/api/bff/me` | `no-store, must-revalidate` | `no-store, must-revalidate` |
| `/sign-in` (public, no member data) | `no-cache, must-revalidate` | `s-maxage=31536000` — cacheable, deliberately |

A reviewer checking this in `npm run dev` would see `no-cache` and conclude the proxy
does nothing. It does; the dev server overrides page headers. Recorded here so that
conclusion is not reached twice.

`/sign-in` is skipped by the proxy on purpose — it carries no member data, and making
the one public screen uncacheable would be a cost with nothing bought.

### Two things found while building it

**`middleware.ts` is deprecated in Next 16.** It warned on every request and pointed at
`proxy.ts`. Migrated rather than shipped — writing new code against a convention the
framework is already deprecating buys nothing and costs a migration later.

**A 500 that was not one.** The first `curl /` returned 500; the server log showed
`GET / 307`. The 500 was a compile-time error page from a request that arrived while the
route was still building. Recorded because the obvious reading — "the redirect is
broken" — was wrong, and chasing it would have cost an hour.

### Mutation-checked

Disabling the "session is not usable" branch — so a stale cookie renders the shell —
fails 2 of 66. That is the branch a cookie-presence check would have got wrong.

**Gate (human-run) — critical-delivery item 3, audit evidence**

| | |
|---|---|
| Command | `npm run lint && npm run build && npm run typecheck && npm test` in `fitforge-web` |
| **Exit code** | *(pending — human-run)* |
| Commit gated | `26a083d` |
| `scope-check-repos` | `PASS phase 8 commit 26a083d (7 file(s))` |
| `git diff --stat` | 7 files changed, 332 insertions(+), 14 deletions(-) |

---

## Phase 9: Profile (US3, US4) — UI phase

**Territory**:

- `fitforge-web/src/**`

### Implementation

- [ ] T088 [P] Add `src/app/api/bff/me/preferences/route.ts`, `me/password/route.ts`, `me/route.ts` (DELETE) per `contracts/member.md` §6.
- [ ] T089 Build `src/app/(app)/profile/page.tsx` to VI-017 through VI-027 — two equal columns at ≥1024px (VI-017), the Preferences card left, the Account card right, **no "My gear" card and no "Export my data" button** (the two declared deviations).
- [ ] T090 Units as a width-to-content segmented control (VI-019); goal, experience and time-zone selects in the orders VI-020 to VI-022 fix.
- [ ] T091 Render VI-023's copy verbatim — "Weeks and streaks are counted in this zone." It is mandatory (K3), and it is the explanation a member needs when a streak breaks at an unexpected hour.
- [ ] T092 Render VI-027's copy verbatim, and cite the same 30-day constant phase 6 defined (T062).
- [ ] T093 "Delete account" is the only destructive-colored control in the product and always sits last (VI-025, VI-026, K4).
- [ ] T094 Extend `src/lib/units.ts` for display conversion, and use it **at render only** — nothing converted is ever sent back (D8, FR-011).

### Tests and the loop

- [ ] T095 [P] Vitest — switching units re-renders every displayed value and sends no measurement to the BFF (VI-028).
- [ ] T096 **Visual Compliance Loop** against `screenshots/11-profile-desktop-{light,dark}.jpg`, both themes, until the deviation table is empty or holds only the declared deviations.
- [ ] T097 Verify the single-column layout below 1024px live (VI-017).

**Gate (human-run)**: as phase 7.

---

## Phase 10: Audit evidence (governance)

**Territory**: this feature's spec directory only — implicitly in territory, declared here as
nothing so that a stray file outside it is a scope-check failure.

- [ ] T098 Complete the Gate blocks above: the command, its exit code, the `scope-check` and `scope-check-repos` verdicts, and `git diff --stat`, for all nine code phases (critical-delivery item 3).
- [ ] T099 File the AI review as `ai-code-review.md` with the Reviewer Provenance block, **including the item-by-item pass over all ten training invariants** (critical-delivery item 2, SC-006). `data-model.md`'s invariant trace is the starting point, not the answer.
- [ ] T100 File Ahmad's `human-pr-review.md` with the `## Review Provenance` block, including its own item-by-item invariant pass (SC-006).
- [ ] T101 Record SC-001 through SC-007 with what settled each: which test, which inspection, which gate. SC-004 is a search of both repositories for a plaintext or reversibly-encoded password, and the search command belongs in the record.
- [ ] T102 Confirm the merge precondition of `plan.md` §6 — FitForge declares `"developers": ["anas.m", "ahmad"]` in `kit-adoption.json`, so the Critical evidence check reads this project as a team and Ahmad's review is the evidence. Blocked on kit feature 013 flowing down; this feature is that feature's SC-001.

**Gate (human-run)**: `pwsh -File scripts/ritual-checks.ps1` on the governance repository.

---

## Not in this feature

Recorded so the next reader does not go looking:

- **Password reset and email verification** — no email delivery exists (`spec.md`,
  Assumptions). The "Forgot?" link is not rendered, and the first feature that emails a
  member also owes address verification.
- **The "My gear" card** — feature 003's catalog (`spec.md`, declared deviations).
- **"Export my data"** — no spec defines its contents or format.
- **Anything computing a day or week boundary** — there is no training data yet. D10 exists
  so feature 007 inherits a validated zone rather than a free-text column.

---

## Phase 1 amendment requests (awaiting an approver)

Constitution I, **Amendment authority**: a change to this file, `plan.md`, `spec.md` or
`contracts/` after approval records who approved it, and **an implementing agent MUST NOT
approve its own amendment**. Three items came up during phase 1. None is applied.

### A1 — T011's method (blocking T011)

**Asks**: change T011 from "the migration applies to an empty database and its `down`
drops both tables" to "the migration's `UpOperations` create exactly `Member` and
`Profile`, and its `DownOperations` drop exactly those two, child first".

**Why**: no SQL Server instance is reachable from the authoring host, and CI has none
either — 001's tests stub the database rather than provisioning one. Applying a migration
needs a real database; reading its operations does not, and it settles the question T011
was written to settle (is the down-path safe and complete?) without one. The alternative,
an in-memory or SQLite provider, would need a package this plan has not approved.

**What is lost**: this would not catch a migration that is valid C# but fails against SQL
Server — a bad check constraint expression, say. That risk moves to the first real
`database update`, and `rollback.md`'s verification list is where it lands.

**Amendment approved by**: anas.m, 2026-09-10

T011's text is amended accordingly, above.

### A2 — `CK_Profile_BirthYear`'s upper bound

**Asks**: `data-model.md` says the constraint bounds `BirthYear` to "1900 to the current
year". As shipped it reads `BETWEEN 1900 AND 2200`.

**Why**: a check constraint is baked into the schema when the migration runs, so
`YEAR(GETDATE())` would freeze to the year of the migration and then quietly drift — by
2027 it would reject a birth year of 2026. A literal that is deliberately loose rejects
the typo class (19, 20260) and leaves plausibility to application validation, which can
compute the current year.

**Amendment approved by**: anas.m, 2026-09-10

`data-model.md`'s `Profile` table is amended to state the shipped bound, carrying the same
approver line.

### A3 — withdrawn. The check caught it, and reverting was the right answer

**What happened**: `dotnet ef migrations add` resolves the `DbContext` through the
**startup** project, so it failed until `Microsoft.EntityFrameworkCore.Design` was
referenced by `FitForge.Api`. That file is not in phase 1's Territory, and
`scripts/scope-check-repos.ps1` said so:

```text
scope-repos: fitforge-api: FAIL phase 1 commit f4cd9e1:
  fitforge-api/src/FitForge.Api/FitForge.Api.csproj not in territory
```

It offered two remediations — revert, or widen the Territory in a governance commit made
before the code commit, with owner approval. **Reverted.** Widening a Territory to fit a
change already made is the retroactive move the rule exists to prevent, and the amendment
that would have licensed it is one the implementing agent may not approve.

The reference is not needed again until phase 3 generates
`AddSessionAndSignInAttempt`, and phase 3's Territory already includes
`fitforge-api/src/FitForge.Api/**`. So T023 carries it: generating that migration means
adding the reference, in a phase where it is declared. **No amendment is owed** — which is
why this request is withdrawn rather than pending.

The phase 1 migration itself is unaffected: it was generated while the reference was
present, and the generated files live in `FitForge.Infrastructure`, which has its own
copy. The build is clean without it.

**This is the first time a machine check, rather than a reviewer, caught a scope error in
FitForge** — and it was in a nested code repository, which is precisely the blindness kit
feature 012 (GAP-016) closed. The check was worth building.

### A4 — how the database-touching tests get a database (blocks T029, T030, T031)

**The problem.** Three of phase 3's tests assert against stored rows: that a live session
resolves and an expired, revoked or soft-deleted-member one does not; that the token is
nowhere in the database; that the expiry slides at most hourly. All three need a database.
There is none — the SQL Server instance is unreachable from the authoring host, and
`fitforge-api`'s CI is `ubuntu-latest` with no service container. This is the same wall
A1 hit, but A1's workaround (read the migration's operations) has no analogue here: these
tests are *about* what persistence does.

**What is actually available**, checked rather than assumed: Docker 28.2.2 is running on
this host, and SQL Server LocalDB (`MSSQLLocalDB`) is installed.

| | Approach | Fidelity | Cost |
|---|---|---|---|
| **a** | **Real SQL Server.** LocalDB or a Docker container locally; an `mssql/server` service container in CI | The engine that ships. Real unique indexes, real FKs, real check constraints, and the migration is genuinely applied | **No package.** CI gains a service container (~30–60s). A local gate run needs Docker or LocalDB up. `.github/workflows/project-gate.yml` must change — **outside phase 3's Territory** |
| b | SQLite in-memory (`Microsoft.EntityFrameworkCore.Sqlite`, test-only) | Relational, so unique indexes and FKs are enforced — but a different engine. `binary(32)` becomes a BLOB, `datetime2(3)` differs, and SQL Server migrations cannot be applied, so the schema under test is **built from the model, not from the migration that ships** | One new package |
| c | `Microsoft.EntityFrameworkCore.InMemory` | No constraints at all | One new package |

**Recommended: (a), and it is not close.** The one thing that makes this feature Critical is
that member isolation must actually hold, and invariant 2 calls a cross-member read "a defect
of the highest severity". Both (b) and (c) test that against something other than the
database that will run it — and (c) would let SC-002's test pass with the unique index
missing, which is worse than having no test, because it reads as evidence. (a) also
retroactively strengthens T011: A1's stated residual was "a migration that is valid C# but
fails against SQL Server", and under (a) that stops being a residual.

**What (a) needs from you, beyond approval of the approach**: phase 3's Territory does not
include `.github/**`, and the CI change lives there. Either widen it in a governance commit
made **before** the CI commit, or give the CI wiring its own phase. I would rather you chose
than have me pick the one that happens to be less work.

**Amendment approved by**: anas.m, 2026-09-10 — **option (a), and the CI wiring gets its
own phase.**

### Why the new phase is numbered 11 and not 4

It runs **next**, before phases 4 through 10, because every one of their tests needs the
database it wires up. It is numbered 11 anyway.

Inserting it as phase 4 would renumber 4->5 ... 10->11, and phase numbers are not only in
this file: they are in comments inside commits that have already been gated.
`MemberConfiguration.cs` cites "the retention service (phase 6)",
`AddMemberAndProfileMigrationTests.cs` the same, `SignInAttempt.cs` likewise. Renumbering
makes each of those false, and correcting them means editing gated code, which means
re-gating phases 1 and 3 to fix a numbering choice.

Worse, `fitforge-api` also carries comments citing **feature 001's** phases 7 and 8
(`UserSecretsIdTests.cs`, `HealthReadyTests.cs`, `DependencyInjection.cs`). A file holding
"phase 8" meaning 001 beside "phase 8" meaning a renumbered 002 is a trap for the next
reader.

So: numeric order is identity, not schedule. The execution order is stated in `plan.md`
§8 and here, and `scripts/scope-check-repos.ps1` grades by the `phase N` token against
that phase's declared Territory, which is unaffected either way.

### T029-T031 move to phase 11

They cannot precede the wiring they depend on. Phase 3 therefore takes its gate covering
the implementation and T031b; phase 11 delivers the fixture, the CI service container, and
the three session tests.

**Stated plainly because it is a real cost**: phase 3 gates without the tests that prove
its central mechanism. The alternative - holding phase 3 ungated and unmerged until phase
11 lands - trades one discomfort for a longer-lived one. The tests arrive one phase later,
and this paragraph is what stops that being quietly forgotten.

**Amendment approved by**: anas.m, 2026-09-10

---

## Phase 11: Test database wiring (runs next — before phases 4–10)

**Added 2026-09-10 by amendment A4. Amendment approved by**: anas.m, 2026-09-10.

**Goal**: give the tests a real SQL Server, locally and in CI, so every later phase asserts
against the engine that ships rather than a stand-in.

**Independent Test**: `dotnet test` passes against a database it created and dropped; the
three session tests moved here pass, and each fails when the behaviour it covers is broken.

**Territory**:

- `fitforge-api/tests/**`
- `fitforge-api/.github/workflows/project-gate.yml`

### Implementation

- [x] T103 Add `tests/FitForge.Api.Tests/SqlServerDatabase.cs` — a fixture that creates a uniquely-named database, applies **the migrations** (not `EnsureCreated`, which builds from the model and would test a schema that never ships), and drops it on dispose.
- [x] T104 Resolve the connection string from `FITFORGE_TEST_SQL`, falling back to LocalDB so a developer on Windows needs no setup. **No credential in source** (constitution VI).
- [x] T105 (was T029) Resolve succeeds for a live session; fails for expired, for revoked, for unknown, and for a session whose member is soft-deleted. Four cases, one answer.
- [x] T106 (was T030) The stored `TokenHash` is not the token, and no column anywhere holds it — asserted against the database, not against the code.
- [x] T107 (was T031) Sliding expiry extends at most once per hour.
- [x] T108 Add an `mssql/server` service container to `.github/workflows/project-gate.yml` and pass `FITFORGE_TEST_SQL` to the test step. The `sa` password comes from a repository **secret**, never inlined — and the workflow fails loudly if the secret is absent rather than silently skipping the database tests.

### Why this earns its own gate

Two things make it more than plumbing. It retires A1's stated residual — T011 could not
catch "a migration that is valid C# but fails against SQL Server", and once migrations are
applied against a real server on every test run, nobody is carrying that risk. And SC-002,
the cross-member read test that is the whole reason this feature is Critical, is worth
something only against a database that actually enforces the unique index and the foreign
key. This phase is the prerequisite for the feature's headline claim being testable at all.

### What the fixture proved on its first run

The 12 session tests pass against **real SQL Server** (LocalDB on the authoring host).
That run applied `AddMemberAndProfile` and `AddSessionAndSignInAttempt` for real, which
**retires A1's residual**: T011 could not catch a migration that is valid C# but invalid
SQL, and both now execute against the engine on every test run. A1's "what is lost" note
is answered rather than merely acknowledged.

**Mutation-checked**, since the whole argument for option (a) was fidelity:

| Mutation | Result |
|---|---|
| the join uses `IgnoreQueryFilters()`, so a soft-deleted member's session resolves | 3 of 12 fail |
| the slide updates the entity but never calls `SaveChangesAsync` | (same run) |

Both reverted; 12 of 12 pass. Note what the first mutation means: it is invariant 2's
failure mode, and the test catches it only because the query filter is a real filter on a
real database.

**CI**: `mcr.microsoft.com/mssql/server:2022-latest` as a service container, with a health
check, and the `sa` password from a repository **secret** — never inlined. A guard step
fails the job with a legible error when the secret is missing, because a container that
silently refuses to start reads like a flake and gets re-run rather than fixed.

> **Action required before this branch's CI can pass**: add `MSSQL_SA_PASSWORD` under
> *Settings → Secrets and variables → Actions* in `anas-mattar/fitforge-api`. Any strong
> value; it protects a throwaway container that lives for the length of one job.

**Gate (human-run) — critical-delivery item 3, audit evidence**

| | |
|---|---|
| Command | `dotnet build --warnaserror && dotnet test` in `fitforge-api` |
| **Exit code** | **0**, run by anas.m, 2026-09-10 |
| Commit gated | `43f22a5` |
| `scope-check-repos` | `PASS phase 11 commit 43f22a5 (3 file(s))` |
| `git diff --stat` | 3 files changed |

The run needs Docker or LocalDB available. LocalDB was used here.

**Phase 11 is complete**, and phase 3 is closed with it — T105–T107 are the tests phase 3
was missing.
