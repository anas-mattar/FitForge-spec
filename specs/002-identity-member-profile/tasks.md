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

- [ ] T077 [P] Vitest — the BFF sign-in route maps 401 to a credential error and an unreachable API to a service error. Every row of the mapping table, as 001 did for health. **Record corrected, phase 12 (F11)**: was `[x]`. `src/lib/__tests__/api-auth.test.ts` imports `postSignIn` from `../auth-transport` — it grades the **transport**, not the route. No test file imports anything under `src/app/api/bff/`. The mapping in `sign-in/route.ts:32-34` can be changed so an unreachable API reports a credential failure — the one outcome `contracts/auth.md:109-111` forbids — with all 76 tests still green. Closes in phase 15.
- [ ] T078 Vitest — the cookie is set with all five attributes, and its value is not readable from a non-`HttpOnly` path. **Record corrected, phase 12 (F11)**: was `[x]`. **This test does not exist.** Nothing under `src/` references `__Host-`, `httpOnly`, `sameSite`, `writeSessionToken`, `clearSessionToken` or `SESSION_COOKIE` outside `src/lib/session.ts` and five route handlers; `session.ts` has no test of any kind. SC-003 rests on T081's one-off inspection and on nothing standing. Closes in phase 15.
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
- [ ] T084 Add `GET /api/bff/me` and have the shell read the member from it server-side. **Record corrected, phase 12 (F11)**: was `[x]`. The route exists; the second half did not happen. `src/app/(app)/layout.tsx:27` calls `getMe()` directly, so `GET /api/bff/me` has no caller and is dead code — and it is also the route that the false "called during a server-side render" rationale in `bff.ts:27-29` was built around (F13). Decide in phase 15 whether the shell uses it or the route goes.
- [ ] T085 Ensure the back button after sign-out does not restore an authenticated view — the response carries no-store, and the shell re-reads the session on every navigation (US1 scenario 5). **Record corrected, phase 12 (F11)**: was `[x]`. **There is no sign-out** — no component in `fitforge-web/src` references the route (F9), so this guards a path no member can take. Two further gaps behind it: `no-store` does not govern Next's client Router Cache, which is what serves a Back after `router.push()` (N21), and `src/proxy.ts` has no tests. Closes in phase 16, after F9 gives it something to guard.

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
| **Exit code** | **0**, run by anas.m, 2026-09-10 |
| Commit gated | `26a083d` |
| `scope-check-repos` | `PASS phase 8 commit 26a083d (7 file(s))` |
| `git diff --stat` | 7 files changed, 332 insertions(+), 14 deletions(-) |

---

## Phase 9: Profile (US3, US4) — UI phase

**Territory**:

- `fitforge-web/src/**`

### Implementation

- [x] T088 [P] Add `src/app/api/bff/me/preferences/route.ts`, `me/password/route.ts`, `me/route.ts` (DELETE) per `contracts/member.md` §6.
- [x] T089 Build `src/app/(app)/profile/page.tsx` to VI-017 through VI-027 — two equal columns at ≥1024px (VI-017), the Preferences card left, the Account card right, **no "My gear" card and no "Export my data" button** (the two declared deviations).
- [x] T090 Units as a width-to-content segmented control (VI-019); goal, experience and time-zone selects in the orders VI-020 to VI-022 fix.
- [x] T091 Render VI-023's copy verbatim — "Weeks and streaks are counted in this zone." It is mandatory (K3), and it is the explanation a member needs when a streak breaks at an unexpected hour.
- [x] T092 Render VI-027's copy verbatim, and cite the same 30-day constant phase 6 defined (T062).
- [x] T093 "Delete account" is the only destructive-colored control in the product and always sits last (VI-025, VI-026, K4).
- [x] T094 Extend `src/lib/units.ts` for display conversion, and use it **at render only** — nothing converted is ever sent back (D8, FR-011).

### Tests and the loop

- [x] T095 [P] Vitest — switching units re-renders every displayed value and sends no measurement to the BFF (VI-028).
- [ ] T096 **Visual Compliance Loop** against `screenshots/11-profile-desktop-{light,dark}.jpg`, both themes, until the deviation table is empty or holds only the declared deviations. **Record corrected, phase 12 (F11)**: was `[x]`. The loop ran and found two real deviations, both fixed — but it missed a third that ships: `PreferencesCard.tsx:113-117` renders a "Height:" row present in neither reference nor VI-019/VI-020, shifting every control below it. A fourth deviation, undeclared, and an **addition** rather than an omission, so the exit condition was not met. It is also permanently `—`, since nothing writes `heightCm` (F14). Closes in phase 16.
- [x] T097 Verify the single-column layout below 1024px live (VI-017).

### Visual Compliance Loop — result

Measured against a running application, both themes, four widths.

| VI | Required | Measured |
|---|---|---|
| VI-017 | two **equal** columns at ≥1024px, 16px gap; one below | `498.009px 498.021px`, gap 16px; one column at 1023/768/390 |
| VI-018 | cards rounded, 1px border, 20px padding, heading 14px semibold | 20px, 14px/600 |
| VI-019 | units segmented **width to content**, 16px/6px padding | `inline-flex`, 158px wide, `6px 16px` |
| VI-020 | goal select 40px, options Hypertrophy → General fitness | 40px; order confirmed |
| VI-021 | experience 40px, Intermediate → Beginner → Advanced | 40px; order confirmed |
| VI-022 | IANA name with UTC offset in parentheses | `Asia/Kuala_Lumpur (UTC+8)` — **after a fix, see below** |
| VI-023 | the copy, verbatim and mandatory | "Weeks and streaks are counted in this zone." |
| VI-024 | Email and Member since, 14px, 8px apart, `19 Aug 2026` | both rows, 8px, `19 Aug 2026` |
| VI-025 | wrapping row of 36px buttons, Delete account last | 36px, `flex-wrap: wrap`, last at every width |
| VI-026 | the ONLY destructive-coloured control | exactly **1** destructive-coloured element on the page |
| VI-027 | the retention copy, verbatim, citing 30 days | rendered verbatim |
| VI-028 | units re-render, nothing stored changes | `167.5 cm` → `5' 6"`; the only request was `{"units":"Imperial"}` |
| both themes | geometry identical, colours differ | geometry byte-identical; card fill differs |

**Two deviations found, both fixed — and a third missed.** The two below are recorded
accurately. **Record corrected, phase 12 (F11, F14)**: the loop's exit condition was not
met. `PreferencesCard.tsx:113-117` renders a "Height:" row that appears in neither reference
capture nor VI-019/VI-020, shifting every control beneath it — an undeclared fourth
deviation, and an *addition* rather than an omission, which is the kind this table was least
likely to catch: the loop compares what the reference shows against what the screen shows,
and an extra element is only visible if you are looking for what should **not** be there. It
closes in phase 16; T096 is reopened above.

**VI-022 — the wrong word.** `Intl`'s `shortOffset` renders `GMT+8` in `en-GB`. The
reference writes `UTC+8`. Same instant, different word, and the reference fixes the word.
Only visible by reading the rendered option, which is what the loop is for.

**A dark-mode defect, and it is feature 001's.** "Change password" measured
`rgb(9, 9, 11)` in **both** themes while every sibling moved to `rgb(250, 250, 250)` —
dark text on a dark card. Cause: a `<button>` carries the user agent's
`color: buttontext`, which does not inherit, so `button.tsx`'s `secondary` and `ghost`
variants — which set no colour — render near-black whatever the theme. Fixed in
`button.tsx` with an explicit `text-foreground`, because the profile screen cannot meet
its dark reference while it stands. Recorded as a 001 defect found by 002: `secondary`
had simply never been used on a screen anyone checked in dark.

**How the final re-check was done, and its limit.** The compiled CSS contains
`.text-foreground{color:hsl(var(--foreground))}` and the class is on the element, which
resolves through the same `--foreground` token measured flipping on `body`, `h2` and
`dd`. The live dark-mode re-probe after the fix did **not** run: the browser tooling
stopped responding. So the fix is proven by construction, not by a second measurement.
A human running the profile screen in dark mode is owed at review, and it is one glance.

### Three environment traps, recorded so nobody re-chases them

| Symptom | Cause | Not a product defect because |
|---|---|---|
| No client component responded — even feature 001's theme toggle | browsing `127.0.0.1` instead of `localhost`; Next blocks cross-origin dev resources and hydration never completes | it works on `localhost`, and in production |
| `text-foreground` had no matching CSS rule | the dev server had not recompiled the stylesheet | the production build contains the rule |
| Phase 8's route-protection tests began failing | vitest's 5s default; the `(app)` layout's module graph grew when this phase added the profile screen, and the cold transform took 5.8s on this filesystem | every assertion still held — it failed on **time**. The budget was raised with the reason recorded in the file, which is what distinguishes it from raising a timeout to hide a hang |

### The API side is stubbed, and that is stated

The screen was graded against a stub API serving a fixed member, not the C# API — whose
own behaviour is covered by its 128 tests. What the loop exercised is the real page, the
real components, the real BFF routes and the real cookie. What it did **not** exercise is
this screen against the real API end to end, and that belongs in the human review.

**Gate (human-run) — critical-delivery item 3, audit evidence**

| | |
|---|---|
| Command | `npm run lint && npm run build && npm run typecheck && npm test` in `fitforge-web` |
| **Exit code** | *(pending — human-run)* |
| Commit gated | `a3ca4b9` |
| `scope-check-repos` | `PASS phase 9 commit a3ca4b9 (13 file(s))` |
| `git diff --stat` | 13 files changed, 883 insertions(+), 5 deletions(-) |

**This is the last implementation phase.** Phase 10 is governance only.

---

## Phase 10: Audit evidence (governance)

**Territory**:

- `specs/002-identity-member-profile/**`

This feature's spec directory only. The entry grants nothing new — that directory is
implicitly in territory on every phase — and this file's header says it "is never
declared". Phase 10 is the documented exception, because it is the only phase with no
other path to name, and a `**Territory**` marker with an empty list is the one shape
`scripts/scope-check.ps1` rejects outright (amendment A5, approved by anas.m 2026-09-12).

- [x] T098 Complete the Gate blocks above: the command, its exit code, the `scope-check` and `scope-check-repos` verdicts, and `git diff --stat`, for all nine code phases (critical-delivery item 3).
- [x] T099 File the AI review as `ai-code-review.md` with the Reviewer Provenance block, **including the item-by-item pass over all ten training invariants** (critical-delivery item 2, SC-006). `data-model.md`'s invariant trace is the starting point, not the answer.
- [ ] T100 File Ahmad's `human-pr-review.md` with the `## Review Provenance` block, including its own item-by-item invariant pass (SC-006).
- [x] T101 Record SC-001 through SC-007 with what settled each: which test, which inspection, which gate. SC-004 is a search of both repositories for a plaintext or reversibly-encoded password, and the search command belongs in the record.
- [x] T102 Confirm the merge precondition of `plan.md` §6 — FitForge declares `"developers": ["anas.m", "ahmad"]` in `kit-adoption.json`, so the Critical evidence check reads this project as a team and Ahmad's review is the evidence. Blocked on kit feature 013 flowing down; this feature is that feature's SC-001.

### T101 — the success criteria, and what settled each

One row per criterion. "Settled by" names something a reader can open. Where nothing
settles a criterion, the row says so rather than borrowing evidence from a neighbour.

| | Settled by | Status |
|---|---|---|
| **SC-001** — one screen, one submission, for a new and for a returning member | Both halves of the segmented control render the same card (T074, T075) and `SignInCard.tsx:64` navigates to `/` on success, where the `(app)` layout admits the session (T082). Register and sign-in are one BFF route each (T072); neither routes away | **Held by inspection, not by a test** — the residual is below |
| **SC-002** — a test proves A cannot read or write B, and fails when the scoping is removed | `MemberScopingTests.cs`: T053 behavioural, T054 structural. The mutation was performed and reverted; "SC-002 is met, and it fails when the scoping is removed" above records it, including which of the four tests did **not** catch it | Met, and its blind spot is documented |
| **SC-003** — nothing script-readable authenticates, and the browser never reaches the API origin | T081's inspection table above, against a real signed-in session on LocalDB. The half that cannot be faked: sign-out revoked 1 of 3 sessions, so the BFF read a cookie the script could not see | Met **on the day it was inspected, and by nothing since** — `ai-code-review.md` F10 found that `src/lib/session.ts` has no test of any kind, so `httpOnly` could be flipped tomorrow and every gate would still exit 0. SC-003 asks for an inspection and got a good one; it has no standing witness |
| **SC-004** — no plaintext or reversibly-encoded password in either repository | The search below | Met, with one class of finding explained rather than waved away |
| **SC-005** — the deviation table is empty at merge, except the three declared | ~~Phase 7's and phase 9's loop results~~ — **this row was wrong when first written, and the AI review corrected it.** `PreferencesCard.tsx:113-117` ships a "Height:" row that appears in neither reference screenshot nor VI-019/VI-020: a **fourth** deviation, undeclared, and an addition rather than an omission (`ai-code-review.md` F14) | **NOT met** |
| **SC-006** — both reviews carry an item-by-item pass over all ten training invariants | Nothing yet: T099 and T100 are those reviews, and neither is filed | **Open** |
| **SC-007** — every phase's gate run by a human, exit code recorded here | Every Gate block — phases 1 through 9 and 11, phase 1 carrying two because it spans both repositories — records a human-run exit code of **0** against a named commit, with the `scope-check-repos` verdict and `git diff --stat` beside it. Phase 9 closed last, on 2026-09-12 | Met |

#### SC-004 — the search, and what it found

Three passes, run once per repository. Per repository because the governance repo does not
descend into the nested code repositories — each is its own git repository and the parent
ignores it, so a single search from the root silently covers one of three:

```text
rg -i -n "(password|pwd|passwd)[A-Za-z]*\s*(=|:|,|\()\s*[\"'`][^\"'`]{4,}[\"'`]"
rg -i -n "(base64|FromBase64|ToBase64|Encrypt|Decrypt|Convert\.To|atob|btoa)"
rg -i -n "(_logger|ILogger|Log\.|Console\.Write|console\.(log|error|warn|info))"
```

| Repository | Password-shaped literals | Reversible encoding | Log statements |
|---|---|---|---|
| `fitforge` (governance) | none | none | n/a — no code |
| `fitforge-api` | 13, **all under `tests/`** | 6, none of a password | 4 |
| `fitforge-web` | none | none | 1 |

**The 13 are test fixtures, and here is why that is not a dodge.** Eleven are the same
synthetic passphrase fed *into* the code under test — a hasher test that cannot supply a
plaintext cannot test a hasher. The other two are `PasswordHash = "not-used-here"`, a
placeholder occupying a non-null column in tests that never verify against it. None is any
person's credential, none is written by product code, and none reaches a log.

**The 6 encodings, checked one at a time rather than counted.** Five are in
`SessionService.cs` — base64url of 32 bytes from `RandomNumberGenerator`, which is how the
session token is *minted*, not how anything is recovered. The sixth,
`MemberPasswordHasher.cs:67`, is base64 of 32 random bytes used as the decoy hash's input,
and its own comment says why that value must stay unguessable. The stored value is
`PasswordHasher<TUser>` v3 — PBKDF2-HMAC-SHA512, 210,000 iterations, 128-bit salt — from
which no plaintext is recoverable.

**The log surface, searched because a password leaks there as easily as into a column.**
Five log statements exist in total: `RetentionService.cs:69`, `RetentionRunner.cs:90`,
`DomainExceptionHandler.cs:36`, `BearerSessionHandler`'s injected factory, and
`fitforge-web/src/app/api/health/route.ts:31`. The only one carrying data carries counts,
and says so in its own comment. Not one takes a password, a token, an email or an
identifier.

#### The SC-001 residual, stated for the human review

`router.push("/")` at `SignInCard.tsx:64` is what makes SC-001 true, and **no test asserts
it**. Searching both suites for an assertion on that navigation returns nothing. What
exists is the code, read; and the SC-003 inspection, which drove sign-in by `fetch` and so
never exercised the submit path that calls it. A regression removing that line would leave
every test green and SC-001 false.

Recorded rather than fixed, because adding a test here is phase 9's work arriving in phase
10 after phase 9 was gated — the scope creep the ritual exists to prevent. It belongs in
Ahmad's review as a decision: accept the residual, or take a phase 12 for it.

### T102 — the merge precondition holds

`plan.md` §6 makes the merge conditional on the Critical evidence check reading FitForge as
a **team**. Confirmed in the two places that has to be true:

- `kit-adoption.json` declares `"developers": ["anas.m", "ahmad"]` — two names, so the team
  arm is selected. Absence would have selected solo, the stricter branch, and demanded the
  second-model substitute plus its cooling-off.
- `scripts/enforcement-pack.ps1` carries `Invoke-CriticalTeamEvidence`, so kit feature 013
  has flowed down and that arm exists here. **This feature is 013's SC-001**; the
  precondition is met rather than pending.

What the team arm will demand of T100's file, so Ahmad fills it once rather than twice:
`human-pr-review.md`, **committed** — it is read from the `HEAD` blob, never the working
tree — carrying a visible `## Review Provenance` section, not wrapped in an HTML comment,
with a filled `**Reviewer**:`, a filled `**Owner**:` that differs from it, and this
sentence verbatim: *This reviewer is not the owner of the feature under review.*

One limit of the check, worth knowing rather than discovering: the roster is **counted,
never matched**. A review naming two people absent from `kit-adoption.json` passes. What
the machine enforces is reviewer ≠ owner; the rest is the human's word.

### What phase 10 still owes

T098 closed on 2026-09-12: anas.m ran phase 9's gate in `fitforge-web` against `a3ca4b9`
and it exited **0**, which was the last *(pending)* cell in this file and with it SC-007.

T099 closed on 2026-09-12: `ai-code-review.md` is filed, produced by **three independent
fresh-context agent sessions** — domain invariants, API identity security, web BFF and
contracts — none of which shared context with the session that wrote the code. Its verdict
is **REQUEST CHANGES: 14 blocking findings**.

**T100 is deliberately not staged.** Ahmad reviews after the blocking findings are
resolved, not before; a human review filed against a diff that is about to change is a
signature on the wrong document. SC-006 needs an item-by-item invariant pass in **both**
reviews, and only one exists.

### What the review changed in this file, and why that matters

Two rows of the T101 table above were **wrong when written**, and the review caught them:

- SC-005 was recorded Met. A fourth, undeclared visual deviation ships (F14).
- SC-003 was recorded Met without noting that nothing tests it (F10).

Both errors have the same shape: they were derived from this file's own phase notes rather
than from the code. That is precisely the failure the review exists to catch, and it lands
on the governance phase as readily as on an implementation one. The rows are corrected
above rather than quietly rewritten — the strike-through is the evidence.

The review's own F11 is the same failure at scale: **five tasks are marked `[x]` against
tests that grade a different module or do not exist** (T077, T078, T084, T085, T096). Every
gate in this feature was run by a human and exited 0, honestly, over a suite that does not
assert what this file says it asserts. Correcting those five records is remediation work
that must happen whether or not the missing tests are written now.

One finding is the owner's alone: **F8-GOV**. `Session` and `SignInAttempt` were exempted
from invariant 8 by a deviation argued against `database-rules.md` — a rulebook, a lower
rung — while `training-invariants.md` has never been amended. Either the columns go in, or
the invariant is amended to name the exception with a recorded approver (constitution
1.1.0). A plan cannot waive a rule of constitutional force, and no agent should close it.

**One number an auditor will query, resolved here so nobody chases it.** Phase 7's block
records `git diff --stat` as 20 files changed while its `scope-check-repos` verdict counts
21 — the only Gate block in this file whose two numbers disagree. Both are right. Commit
`ea76672` moves `src/app/page.tsx` to `src/app/(app)/page.tsx`. `git show --stat` prints a
rename as one entry; `Get-CommitPaths` in `scripts/scope-lib.ps1` counts **both sides on
purpose** — its own comment says "renames contribute both sides" — because a rename out of
territory and a rename into it are different questions, and a check that saw only the
destination would let a file be moved out of its phase's territory unnoticed. So 20 and 21
are the same commit, counted for two different purposes. Neither record is edited: they are
gated audit evidence, and this paragraph is the reconciliation.

**Gate (human-run)**: `pwsh -File scripts/ritual-checks.ps1` on the governance repository.

| | |
|---|---|
| Command | `pwsh -File scripts/ritual-checks.ps1` |
| **Exit code** | **1**, run by anas.m, 2026-09-12 — **6 of 7 members green; the sole failure is `CriticalEvidence`** |
| Commit gated | `5c740f1` |
| `scope-check` | `PASS phase 10 commit 5c740f1 (2 file(s))` |
| `git diff --stat` | 2 files changed, 391 insertions(+), 4 deletions(-) |

One run gates this phase and phase 12: `ritual-checks` grades the **branch**, not a commit.
The member breakdown, and why the exit code cannot be 0 on a Critical branch before merge, are
recorded under phase 12's Gate block. Neither phase can carry a 0 until T100 exists.

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

## Phase 10 amendment request

### A5 — phase 10's Territory block declared no entries, and the check rejects that

**Found by running the check, not by reading the file.** `pwsh -File scripts/scope-check.ps1`
against the first attempt at the phase 10 records commit:

```text
scope-check: FAIL phase 10 commit 8cc830b: **Territory** declared for phase 10 in
  specs/002-identity-member-profile/tasks.md but the entry list is empty
  (declare the paths, or remove the marker)
```

`8cc830b` is not reachable on this branch. It was withdrawn (`git reset --soft`) the moment
the check failed, rather than left in history as a failing phase commit — `scope-check.ps1:276`
grades **every** commit from the merge base to HEAD, so one failing commit fails the branch
for the rest of its life. "Commit now, fix after" was never available.

**Why it failed.** Phase 10's Territory read *"this feature's spec directory only —
implicitly in territory, declared here as nothing so that a stray file outside it is a
scope-check failure."* The intent was right and the check agrees with it: `Get-Territory`
adds `specs/$FeatureBranch/**` as an implicit entry for every phase. But a `**Territory**`
marker with **zero entries** is read as a malformed declaration, not an empty one, and
fails closed (`scope-check.ps1:220-223`). Prose where the parser wants a list. Phase 10 is
the only phase in this feature that names no paths of its own, which is why nine phases and
eleven gate runs never met it.

**The two candidate fixes, and why they are not equivalent.**

- **(a) Declare the directory explicitly** — one entry, `specs/002-identity-member-profile/**`.
- **(b) Remove the `**Territory**` marker** — no marker, no parse.

(b) looks tidier and is worse. With no marker, `scope-check.ps1:224-227` emits
`WARN … no territory declared` and **returns true immediately**, short-circuiting the stray
check entirely. Phase 10 would stop being graded at all — the precise opposite of what the
original wording was reaching for. The block said it wanted a stray file to be a failure;
only (a) delivers that.

**Not widening.** The spec directory is already in territory implicitly on every phase, so
(a) grants nothing the phase did not have. It is still an amendment to an approved
`tasks.md`, and Territory is exactly the field where a retroactive edit is the abuse the
rule exists to prevent — which is why it was recorded here and applied by nobody until an
approver signed it (constitution I, amendment authority; the A3 precedent).

**One inconsistency it creates, stated rather than hidden.** This file's header says the
feature's own spec directory "is implicitly in territory and is never declared". After (a)
that sentence has exactly one exception, and it is phase 10. The header is left alone: the
narrower fix is to record the exception where it lives rather than to amend a general
statement that is true of every other phase.

**Amendment approved by**: anas.m, 2026-09-12 — **option (a)**.

**Applied**: phase 10's Territory block above now declares `specs/002-identity-member-profile/**`.
This amendment is committed **before** the phase 10 records commit, because
`scope-check.ps1:194` resolves the declaration from the commit's **parent** — a commit can
never declare its own territory. That is the same ordering A3 was reverted for not having.

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

**One commit of this phase was never graded, and it is worth knowing why.** CI reports
`scope-check: WARN commit bd6c737: no territory declared for phase 11`. The declaration
above is not at fault — it parses. `bd6c737` is the **A4 amendment commit that created
phase 11**, and its subject carries the token `phase 11`, so the check attributed it to a
phase whose Territory did not yet exist *in its parent*, and warned instead of grading. The
rule to carry forward: **a commit that declares a phase must not carry that phase's token
in its subject**, or it grades itself against a declaration that does not exist yet. A5's
commit avoided this by naming no phase.

---

## Remediation amendment request (awaiting an approver)

### A6 — how the 14 blocking findings get fixed, and the one decision they wait on

`ai-code-review.md` returned **REQUEST CHANGES**. Remediation needs phases, phases need
Territory, and Territory must be committed before the phase commit that uses it
(`scope-check.ps1:194` — the lesson of A5, and of `bd6c737` above). So the shape is proposed
here, once, rather than five times over.

#### Part 1 — the F8-GOV decision (invariant 8 vs `Session` and `SignInAttempt`)

**Recommended: amend `modules/training/training-invariants.md`, narrowly — do not add the
columns.**

The reviewer's own judgement was that the engineering argument in `Session.cs:12-22` is
sound and the *instrument* was wrong. Both halves look right to me:

- A session's external identifier **is its token**. Giving it a `PublicId` mints a second
  addressable handle to an authentication artifact — one more thing to leak, log or
  enumerate — and buys nothing. Invariant 8 exists to give traceable identity to records
  people refer to; nobody refers to a session row.
- `SignInAttempt` is a counter, not a record. Its own type comment says so, retention prunes
  it, and `CreatedBy` on a row written by a **failed, unauthenticated** attempt has no
  honest value — "by whom" is exactly what is unknown at that moment.
- Complying costs a migration on two tables plus a GUID and a unique index on the
  highest-churn table in the schema, to populate columns no query will read.

**What the amendment must contain, so it closes the hole instead of widening it**: a named
exemption for entities that are neither externally addressable nor member-facing records —
authentication sessions and rate-limit counters — **and** a requirement that each exempt
entity states its exemption in `data-model.md`. Without that second half this becomes the
escape hatch F8-GOV warned about.

**This one is the owner's alone.** It changes a document of constitutional force, it needs a
recorded approver under constitution 1.1.0, and the implementing agent may not approve it.
If it is refused, phase 14 below gains a migration and its estimate changes.

#### Part 2 — five remediation phases

Ordered so the record stops being false before anything is built on it.

**Phase 12 — the record.** Territory: `specs/002-identity-member-profile/**`. Closes **F11**:
correct the five task records marked `[x]` against tests that grade a different module or do
not exist (T077, T078, T084, T085, T096). No code. First, because every later phase is
graded against this file and it currently overstates what nine phases delivered.

**Phase 13 — the throttle.** Territory: `fitforge-api/src/**`, `fitforge-api/tests/**`,
`fitforge-web/src/**`. Closes **F1** (both halves — the BFF forwards the address, the API
validates it against known proxies), **F2** (serialize check-and-record), **F3** (throttle
register), **F6** (throttle the `/me` re-authentications) and **N1** (`Retry-After` reads
the wrong row). First code phase, because F1 is the one defect that takes the product down
for every member at once.

**Phase 14 — the contracts.** Territory: `fitforge-api/src/**`, `fitforge-api/tests/**`.
Closes **F4** (sign-out idempotence), **F5** (`/problems/no-session`) and **F7** (emit a UTC
offset, and assert the wire shape). Each is a divergence from a document approved under
constitution VII *before* implementation, so each is fixed in the code rather than by
editing the contract — unless the owner rules otherwise, per finding.

**Phase 15 — the web perimeter and its missing tests.** Territory: `fitforge-web/src/**`.
Closes **F12** (compare the whole origin, scheme included), **F13** (refuse Origin-less
mutating requests), **F10** (the cookie's five attributes get the test T078 claimed) and the
untested BFF route handlers named in the review's coverage section.

**Phase 16 — the UI.** Territory: `fitforge-web/src/**`. Closes **F9** (there is no way to
sign out) and **F14** (the undeclared "Height:" row). F9 needs a second owner decision: the
reference screenshots have no sign-out control either, so either the design gains one or
`spec.md`'s US1 acceptance scenario 5 is amended.

Each phase's task list is written when that phase is claimed, not now. What this request
settles is the **Territory**, which has to exist before the phase commit — and, per
`bd6c737` above, the commit that applies this amendment will carry **no phase token**.

#### Not in remediation

The 25 non-blocking findings, except N1, which rides along with the throttle. They are
recorded in `ai-code-review.md` and belong to whoever next opens those files. Folding them
in would turn five phases into ten and blur what "the blocking findings are fixed" means.

**Amendment approved by**: anas.m, 2026-09-12 — **both parts**: the invariant is amended
(not the columns added), and all five phases are declared now.

**Applied**: `modules/training/training-invariants.md` §8 carries the infrastructure-rows
exemption with its two conditions and the approver line; `data-model.md` states the exemption
for `Session` and for `SignInAttempt` (condition 1) and its invariant-8 trace row no longer
claims the invariant is satisfied; phases 12–16 are declared below.

---

## Phase 12: The record (governance)

**Declared 2026-09-12 by amendment A6. Amendment approved by**: anas.m, 2026-09-12.

**Goal**: make this file true before anything is built on it. Closes **F11**.

**Independent Test**: no task in this file is marked `[x]` against a test that does not exist
or that grades a module other than the one the task names.

**Territory**:

- `specs/002-identity-member-profile/**`

Closes: **F11** — T077, T078, T084, T085 and T096 are marked `[x]` against tests that grade a
different module or do not exist. No code. First, because every later phase is graded against
this file.

### Tasks

- [x] T108 Return T077, T078, T084, T085 and T096 to `[ ]`, each carrying **what is actually
  true**, the finding that found it, and the phase that closes it. The task text itself is
  untouched: the requirement was never wrong, only the claim that it was met.
- [x] T109 Correct every completion claim that the five falsify — the "Phase N is done" lines
  and the phase summaries that counted those tasks as delivered.
- [x] T110 Leave the Gate blocks alone. Each records a command a human ran and the code it
  exited with, and every one of those runs really happened. A gate certifies that the suite
  passed, never that the suite was sufficient — editing them would replace a true record with
  a different one.

### What this phase does not do

It writes no test and fixes no code. The five tasks stay open until phases 15 and 16 close
them. The point is that the file now says so.

**The distinction T110 rests on, because it is the one worth carrying forward.** Eleven gates
were run by a human and exited 0. Not one of those records is false: `npm test` really did
pass. What was false was this file's claim about *what those tests covered*. A green gate is
evidence that the suite passed, and evidence of nothing else — and the way that becomes
dangerous is exactly this: a suite that grades a layer below the one the record names, with
the gap recorded nowhere. Correcting the claim and preserving the gate is what keeps both
facts true at once.

### What was corrected

| Task | Was | Is |
|---|---|---|
| T077 | `[x]` — the BFF sign-in route's mapping is tested | `[ ]` — the test grades `auth-transport`, not the route; no test imports anything under `src/app/api/bff/` |
| T078 | `[x]` — the cookie's five attributes are tested | `[ ]` — the test does not exist; `src/lib/session.ts` has no test of any kind |
| T084 | `[x]` — the route exists and the shell reads from it | `[ ]` — the route exists; the shell calls `getMe()` directly, so it has no caller |
| T085 | `[x]` — Back after sign-out is guarded | `[ ]` — no sign-out control exists, so there is no path to guard |
| T096 | `[x]` — the loop's exit condition was met | `[ ]` — an undeclared fourth deviation ships |

Phase 9's Visual Compliance Loop result is corrected in place for the same reason. No Gate
block was touched (T110).

**Gate (human-run)**: `pwsh -File scripts/ritual-checks.ps1` on the governance repository.

| | |
|---|---|
| Command | `pwsh -File scripts/ritual-checks.ps1` |
| **Exit code** | **1**, run by anas.m, 2026-09-12 — **6 of 7 members green; the sole failure is `CriticalEvidence`** |
| Commit gated | `81c79ee` |
| `scope-check` | `PASS phase 12 commit 81c79ee (1 file(s))` |
| `git diff --stat` | 1 file changed, 65 insertions(+), 6 deletions(-) |

**This is the first gate in the feature that could not exit 0, and the reason is structural.**
Phases 1–9 and 11 gate on `dotnet test` or `npm test` in a code repository, and all exited 0.
Phases 10 and 12 gate on `ritual-checks` in the governance repository, whose `CriticalEvidence`
member demands `human-pr-review.md` — an artifact that cannot exist until Ahmad reviews at the
end (Definition of Done item 6 is "once per feature, **at merge**"). That is GAP-022 exactly,
and it is why the exit code is recorded beside the member breakdown rather than alone: on this
branch the summary line carries no information, and only the seven member lines do.

What the run established, beyond the one expected failure:

| Member | Result |
|---|---|
| `doc-lint` | OK — 42 docs, every referenced path resolves |
| `enforcement-pack` | **FAIL** — `CriticalEvidence` only. Two non-blocking `PhaseSizeWarning`s on `ae00e4e` and `d067570`, both planning-stage commits, neither from this phase |
| `scope-check` | OK — every phase commit PASS, including `PASS phase 12 commit 81c79ee`; the three amendment commits correctly read as *not applicable* |
| `scope-repos` | OK — all 11 code-phase commits PASS across both repositories |
| `digests` | OK — 5 fresh, 73 markers |
| `roadmap-claims` | OK |
| `verify-kit` | OK — 2 developers declared, so the **team** evidence rule applies |

**Whether this closes phase 12 is the owner's call, and it is not an agent's to make.** The
evidence is that the phase's own work is clean on every member that can speak to it, and that
the one red member is red for a reason that has nothing to do with this phase and will stay red
until T100. The `scope-check: WARN commit bd6c737` line is the historical phase-11 warning
explained above, unchanged by this phase.

Expect `enforcement-pack FAIL` on `CriticalEvidence` — `human-pr-review.md` still does not
exist and cannot until Ahmad reviews at the end (GAP-022). Six of seven members should be
green; read the member lines, not the summary.

## Phase 13: The throttle (US1, US2)

**Declared 2026-09-12 by amendment A6. Amendment approved by**: anas.m, 2026-09-12.

**Goal**: make FR-016 true in production rather than only in the test harness.

**Independent Test**: a failed sign-in from one address does not raise the 429 threshold for
any other address, asserted against the header the **BFF actually sends**; and parallel
attempts against one address are throttled as strictly as serial ones.

**Territory**:

- `fitforge-api/src/**`
- `fitforge-api/tests/**`
- `fitforge-web/src/**`

Closes: **F1** (both halves — the BFF forwards the source address, the API validates it
against configured proxies), **F2** (serialize check-and-record), **F3** (throttle register),
**F6** (throttle the `/me` re-authentications), **N1** (`Retry-After` reads the wrong row).
First code phase, because F1 is the one defect that takes the product down for every member
at once.

### Baseline gate on untouched code (`CLAUDE.md` workflow step 2)

Run by anas.m, 2026-09-12, before this phase changes anything. Both code repositories were
clean and at the tips left by phase 9.

| Repository | Command | Commit | Exit code |
|---|---|---|---|
| `fitforge-api` | `dotnet build --warnaserror && dotnet test` | `2233d8e` | **0** |
| `fitforge-web` | `npm run lint && npm run build && npm run typecheck && npm test` | `a3ca4b9` | **0** |

**What this baseline does and does not establish.** It establishes that A6 and phase 12 —
both governance-only — changed no code and broke nothing, so any suite failure during phase 13
belongs to phase 13. It establishes nothing about whether the suites are *sufficient*: F1's
own defect is invisible to a green `dotnet test`, because `CredentialEndpointTests.cs:42-46`
injects the `X-Forwarded-For` header the production BFF never sends. A baseline is a
comparison point, not a verdict — which is the distinction phase 12 was spent on.

### Tasks

**F1 — the source address, end to end**

- [x] T111 `SourceAddress` (api): the caller is the connection's own address, unless that
  address is a configured proxy, in which case it is the last `X-Forwarded-For` entry — and
  **unknown** when a trusted proxy names nobody, names something unparseable, or names
  another trusted proxy. Unknown is `null`, never a placeholder: the whole of F1 is that
  "I do not know who this is" was spelled the same way for every caller in the product.
- [x] T112 `Security:TrustedProxies` (api), required and validated at startup beside the
  salt, with a message that spends its length on the failure mode rather than the syntax.
  It has no safe default — an empty list silently makes every caller's address the BFF's —
  so it is refused at startup rather than defaulted.
- [x] T113 The BFF reads the caller from its own inbound `x-forwarded-for` — the **last**
  entry, the one the nearest proxy wrote — and forwards it on sign-in, register,
  `/me/password` and `DELETE /me`. Omitted, never empty, when unknown.
- [x] T114 (web) The test the review asked for by name: assert the header **leaves the
  BFF**, and is absent when the address is unknown. `src/lib/__tests__/source-address.test.ts`.
- [x] T115 (api) The trust boundary, asserted on the stored bucket key: an untrusted peer
  varying the header lands in **one** bucket, a trusted proxy's two addresses land in two,
  and a proxy naming nobody is indistinguishable from a proxy naming itself.

**F2 and N1 — the decision**

- [x] T116 `SignInThrottle.TryRecordAsync` records and decides in one batch, before any
  hashing. `RetryAfterAsync` and `RecordFailureAsync` are gone: they were the two halves of
  a check-then-act, and keeping either as a public method would leave the defect available.
- [x] T117 (api) Sixty-four simultaneous attempts against one address get **exactly ten**
  through, asserted against both the returned decisions and the rows.
- [x] T118 `Retry-After` reads the *n*-th most recent attempt rather than the *n*-th oldest,
  with a fixed clock placing ten failures a minute apart so the two answers differ by nine
  minutes. Floored at one second.

**F3 — register**

- [x] T119 Register consults the throttle on the same two buckets before any hashing, and
  answers the 429 `contracts/auth.md` §2 has listed since before implementation. An existence
  check moved ahead of the hash, so a probe that will be told 409 no longer costs the server
  210,000 iterations; the unique index remains the arbiter under a race.
- [x] T120 (api) The eleventh probe against one address is 429 with a `Retry-After`, and a
  successful registration leaves no row in either bucket.

**F6 — the `/me` re-authentications**

- [x] T121 `POST /me/password` and `DELETE /me` consult and record on the member's own email
  bucket, and clear it the moment the current password verifies — so a member who proves
  their password and then picks three rejected new ones has made one successful attempt, not
  four failed ones.
- [x] T122 (api) Ten wrong guesses then a 429, on both routes, with the account still present.

### What this phase found on its way through

Three things worth keeping, because each was invisible until something ran.

**The atomic statement deadlocked, and the obvious fix did not help.** Counting both buckets
under `UPDLOCK, HOLDLOCK` in one `WHERE` deadlocked under T117 within seconds. Splitting the
counts into ordered statements — so the optimizer could not reorder them — did not help
either. The deadlock graph from `system_health` said why: the cycle was three processes deep
and **entirely inside `IX_SignInAttempt_Email_At`**, every lock a `RangeS-U`. A key-range
lock is taken per key as a scan walks a range, so concurrent scans over a range other
transactions are inserting into acquire locks in an order nobody controls. No statement
ordering fixes that, because the resources are not two things — they are however many keys
the window holds. `sp_getapplock` replaces them with exactly two named resources, always
taken email-first, and a cycle then cannot be formed rather than being unlikely.

**T117 was verified against a broken implementation before it was trusted.** With the two
locks disabled the same test admitted 13, 17, 20, 23, 25, 31, 37, 40, 41 and 45 attempts
across twelve rounds; with them it admitted exactly 10 in all twelve. `plan.md` D7's rule —
a test that passes both ways is not evidence — applied to a concurrency test, where it is
easiest to write one that can only pass.

**A stale build briefly made the fix look broken.** Restoring the file after that experiment
preserved the backup's timestamp, MSBuild judged the project up to date, and three runs
graded a binary whose source no longer existed. It read exactly like a partially-working
lock. Worth recording next to F11: a green or red suite is evidence about *what was built*,
which is not always what is on disk.

### What this phase deliberately does not close

**Unlimited successful registration.** F3's second failure scenario is that every call to
`/auth/register` reaches a 210,000-iteration hash unauthenticated. The throttle now caps the
**failing** half of that completely — probes against existing addresses cost an attacker ten
per address and thirty per source, and no longer cost the server a hash at all. It does not
cap *successful* registrations, because `contracts/auth.md` §6 says both windows are
"counted on failed attempts" and a success is not a failure.

Capping it needs §6 amended to count register attempts regardless of outcome — **an owner
decision, not an agent's**, and one with a real cost: thirty registrations from one office
in fifteen minutes would then throttle the thirty-first. Recorded here rather than taken.

**Gate (human-run)**: one per code repository. Critical forbids batching, so both are run
and confirmed separately (`docs/sdlc/critical-delivery.md` item 4).

| | |
|---|---|
| Command | `dotnet build --warnaserror && dotnet test` in `fitforge-api` |
| **Exit code** | **0**, run by anas.m, 2026-09-12 — `total: 144, failed: 0, succeeded: 144`; `Build succeeded in 142.5s` |
| Commit gated | `4cf4549` |
| `scope-check-repos` | `PASS phase 13 commit 4cf4549 (12 file(s))` |
| `git diff --stat` | 12 files changed, 1318 insertions(+), 89 deletions(-) |

| | |
|---|---|
| Command | `npm run lint && npm run build && npm run typecheck && npm test` in `fitforge-web` |
| **Exit code** | **0**, run by anas.m, 2026-09-12 — `Test Files 10 passed (10)`, `Tests 86 passed (86)` |
| Commit gated | `121453b` |
| `scope-check-repos` | `PASS phase 13 commit 121453b (11 file(s))` |
| `git diff --stat` | 11 files changed, 266 insertions(+), 12 deletions(-) |

The governance commit `32f0c4b` is covered by the branch-level `ritual-checks` run recorded
below, not by a gate of its own.

**Branch checks after this phase** — `pwsh -File scripts/ritual-checks.ps1`, exit **1**,
**6 of 7 members green, `CriticalEvidence` the sole failure**. Unchanged in kind from phases
10 and 12: `human-pr-review.md` cannot exist until T100 (GAP-022). What this run adds is the
first grading of real code since the remediation began —

| Member | Result |
|---|---|
| `doc-lint` | OK — 42 docs, every referenced path resolves |
| `enforcement-pack` | **FAIL** — `CriticalEvidence` only. Two non-blocking `PhaseSizeWarning`s on `ae00e4e` and `d067570`, both planning-stage commits; this phase's governance commit triggered neither |
| `scope-check` | OK — `PASS phase 13 commit 32f0c4b (2 file(s))` |
| `scope-repos` | OK — **`PASS phase 13` in both code repositories, 23 files of real code graded against the Territory declared in this file** |
| `digests` | OK — 5 fresh, 73 markers |
| `roadmap-claims` | OK |
| `verify-kit` | OK — 2 developers declared, so the team evidence rule applies |

The `WARN commit bd6c737` line is the historical phase-11 warning explained above, unchanged
by this phase.

### One record corrected

`fitforge-web` commit `121453b`'s message says *"96 tests pass locally"*. The suite has
**86**, and had 86 when that line was written; the number was miscopied, not measured. It is
corrected here rather than by rewriting a pushed commit — the same treatment phase 12 gave
the five false task records, and for the same reason: the fix for an untrue record is a true
one beside it, not a quieter version of the original. Nothing else in that message is
affected, and the gate above is the count that governs.

## Phase 14: The contracts (US1, US2, US3)

**Declared 2026-09-12 by amendment A6. Amendment approved by**: anas.m, 2026-09-12.

**Goal**: make the API behave as the contracts approved under constitution VII say it does.

**Independent Test**: a second sign-out returns 204; every 401 on an authenticated path
carries `type: /problems/no-session`; a serialized instant carries a UTC offset, asserted on
the wire rather than in the model.

**Territory**:

- `fitforge-api/src/**`
- `fitforge-api/tests/**`

Closes: **F4** (sign-out idempotence), **F5** (`/problems/no-session`), **F7** (UTC offset on
the wire). Each is a divergence from a document approved *before* implementation, so each is
fixed in the code rather than by editing the contract — unless the owner rules otherwise, per
finding.

## Phase 15: The web perimeter, and the tests it never had

**Declared 2026-09-12 by amendment A6. Amendment approved by**: anas.m, 2026-09-12.

**Goal**: make the CSRF defence hold against the attack it was written for, and put a test
under the cookie that SC-003 rests on.

**Independent Test**: a request whose `Origin` differs only in scheme is refused; a mutating
request with no `Origin` is refused; and flipping `httpOnly`, the `__Host-` prefix, `secure`,
`sameSite` or `path` in `src/lib/session.ts` fails the suite.

**Territory**:

- `fitforge-web/src/**`

Closes: **F12** (compare the whole origin, scheme included), **F13** (refuse Origin-less
mutating requests), **F10** (the cookie test T078 claimed), and the untested BFF route
handlers named in the review's coverage section.

## Phase 16: The UI (US1, US3) — UI phase

**Declared 2026-09-12 by amendment A6. Amendment approved by**: anas.m, 2026-09-12.

**Goal**: give the member the sign-out US1 scenario 5 promises, and stop rendering a control
the references do not have.

**Independent Test**: a signed-in member can end their session from the UI at every
breakpoint, and pressing Back afterwards does not restore an authenticated view; the
Preferences card matches `screenshots/11-profile-desktop-{light,dark}.jpg` with only the
declared deviations.

**Territory**:

- `fitforge-web/src/**`

Closes: **F9** (there is no way to sign out) and **F14** (the undeclared "Height:" row).

**Blocked on one owner decision before it can be claimed**: the reference screenshots have no
sign-out control, so either the design gains one — a VI item, and `spec.md`'s visual section
is amended to carry it — or US1 acceptance scenario 5 is amended to drop the promise. F9
cannot be closed without choosing, and an agent must not choose for you.
