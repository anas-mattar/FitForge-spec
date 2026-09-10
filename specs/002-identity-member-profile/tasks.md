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

- [ ] T021 Add `src/FitForge.Domain/Members/Session.cs` and `SignInAttempt.cs` per `data-model.md`, including the three approved deviations from `database-rules.md` (D3) — each one commented with the plan decision that approved it, so the next reader finds the reason and not just the absence.
- [ ] T022 Add `SessionConfiguration.cs` and `SignInAttemptConfiguration.cs` — `UQ_Session_TokenHash`, `IX_Session_MemberId`, `IX_SignInAttempt_Email_At`, restrict FKs.
- [ ] T023 Generate migration `AddSessionAndSignInAttempt`. One migration per phase (`database-rules.md`).
- [ ] T024 Add `src/FitForge.Api/Features/Identity/SessionService.cs` — issue (256 random bits from `RandomNumberGenerator`, store `SHA-256` only), resolve by hash, revoke one, revoke all-but-one, revoke all. The token is returned to the caller exactly once and never read back from storage (D3).
- [ ] T025 Add `src/FitForge.Api/Hosting/Authentication/BearerSessionHandler.cs` — turns `Authorization: Bearer <token>` into the current member, or 401. Expired, revoked, unknown and "belongs to a soft-deleted member" all produce the same 401 (`contracts/auth.md` §5).
- [ ] T026 Add `CurrentMember` as the only way a handler learns who is asking. There is no other accessor, so D7 has one place to be right.
- [ ] T027 Implement sliding expiry — extend to now + 14 days on resolve, at most once per hour, so the hot path is not a write per request.
- [ ] T028 Map `GET /api/v1/auth/session` per `contracts/auth.md` §5.

### Tests

- [ ] T029 [P] `FitForge.Api.Tests` — resolve succeeds for a live session; fails for expired, for revoked, for unknown, and for a session whose member is soft-deleted. Four cases, one message.
- [ ] T030 `FitForge.Api.Tests` — the stored `TokenHash` is not the token, and no column anywhere holds the token (D3). Asserted against the database, not against the code.
- [ ] T031 `FitForge.Api.Tests` — sliding expiry extends at most once per hour.

**Gate (human-run)**: as phase 1.

---

## Phase 4: Credentials (US1)

**Territory**:

- `fitforge-api/src/FitForge.Api/**`
- `fitforge-api/appsettings.json`
- `fitforge-api/tests/**`

### Implementation

- [ ] T032 Map `POST /api/v1/auth/register` per `contracts/auth.md` §2 — validate with D1, normalize with T004, create `Member` **and** `Profile` in one transaction (`data-model.md`: no read path handles a missing profile), issue a session, return 201.
- [ ] T033 Handle the duplicate email as 409, and rely on `UQ_Member_NormalizedEmail` to be the actual arbiter — two concurrent registrations of the same address must produce one member and one 409, not two members.
- [ ] T034 Map `POST /api/v1/auth/sign-in` per `contracts/auth.md` §3 — one message and one status for unknown email, wrong password and soft-deleted member.
- [ ] T035 Implement the **decoy hash path** (D6): when no member is found, verify the supplied password against the startup decoy so both paths do the same work.
- [ ] T036 Implement the throttle (D5, `contracts/auth.md` §6) — two 15-minute fixed windows, 10 per normalized email and 30 per source, both answering 429 with `Retry-After`. **Attempts against addresses that do not exist are counted identically**; this is the requirement, not an implementation detail.
- [ ] T037 Add the source-address salt as a configuration **name** with an empty value in `appsettings.json`, documented in the repository README. No secret enters source (constitution VI).
- [ ] T038 Map `POST /api/v1/auth/sign-out` — revoke server-side, 204, idempotent (FR-006).
- [ ] T039 Audit every log statement added in this phase against D12: no password, no token, no hash, no source address, no internal `Id`.

### Tests

- [ ] T040 `FitForge.Api.Tests` — register, then sign in, then resolve the session. The whole of US1 in one test.
- [ ] T041 [P] `FitForge.Api.Tests` — unknown email and wrong password return byte-identical bodies and the same status (FR-004).
- [ ] T042 `FitForge.Api.Tests` — the decoy path **calls the hasher**. Asserted through a counting hasher, not through wall-clock timing: a timing assertion in CI is a flaky test, not a security control (D6).
- [ ] T043 [P] `FitForge.Api.Tests` — registering an email differing only in case, or by surrounding whitespace, is refused as a duplicate (FR-001, spec Edge Cases).
- [ ] T044 `FitForge.Api.Tests` — the throttle fires on the 11th failure for an email **that does not exist**, and answers 429 exactly as it does for one that does (D5). This is the oracle test; without it the feature's headline defence is untested.
- [ ] T045 `FitForge.Api.Tests` — a successful sign-in clears the email bucket and leaves the source bucket intact.
- [ ] T046 `FitForge.Api.Tests` — sign-out revokes server-side: the token fails on the next resolve, and a second sign-out with the same token is still 204.

**Gate (human-run)**: as phase 1.

---

## Phase 5: The `/me` surface (US3, US4)

**Territory**:

- `fitforge-api/src/FitForge.Api/**`
- `fitforge-api/tests/**`

### Implementation

- [ ] T047 Map `GET /api/v1/me` per `contracts/member.md` §1 — member plus profile, every profile field nullable, no internal `Id` in the payload (invariant 8).
- [ ] T048 Map `PATCH /api/v1/me/preferences` §2 — units, goal, experience, time zone; absent means unchanged; `null` is not accepted for any of the four.
- [ ] T049 Validate the time zone with `TimeZoneInfo.FindSystemTimeZoneById` and return 422 with a member-legible message when it does not resolve (D10). Never a silent fall back to UTC.
- [ ] T050 Map `POST /api/v1/me/password` §3 — verify the current password, apply D1 to the new one, revoke **every other** session, keep the presented one (FR-013).
- [ ] T051 Map `DELETE /api/v1/me` §4 — re-authenticate with the password, then in one transaction soft-delete the member and everything they own and revoke every session including the presented one (FR-014).
- [ ] T052 Confirm by inspection that no path under `/me` declares a route parameter, query parameter, or body field naming a member (D7). T054 turns this from a habit into a check.

### Tests

- [ ] T053 **[D13-3]** `FitForge.Api.Tests` — SC-002: two members; A's session with B's `PublicId` supplied in a body, a query string and a header returns A's data every time. Then remove the scoping and confirm the test **fails** — a test that passes both ways is not evidence.
- [ ] T054 **[D13-2]** `FitForge.Api.Tests` — enumerate mapped endpoints under `/me` and assert none declares a member-naming parameter. This is the test most likely to be deleted by someone who finds it annoying; that is the argument for it.
- [ ] T055 **[D13-4]** `FitForge.Api.Tests` — changing `units` leaves every measurement column byte-identical (FR-011, invariant 4, VI-028).
- [ ] T056 [P] `FitForge.Api.Tests` — password change: the old password stops working, the new one works, other sessions are dead and the presented one survives.
- [ ] T057 [P] `FitForge.Api.Tests` — a wrong current password refuses the change and leaves the existing password working.
- [ ] T058 `FitForge.Api.Tests` — after deletion, sign-in returns the same 401 as a wrong password: a deleted account is not discoverable (FR-014).
- [ ] T059 [P] `FitForge.Api.Tests` — an unresolvable time-zone identifier is a 422, not a silent UTC (spec Edge Cases).
- [ ] T060 `FitForge.Api.Tests` — two concurrent password changes: one wins, the other is refused, and the account is never left with neither password working.

**Gate (human-run)**: as phase 1.

---

## Phase 6: Retention (US4)

**Territory**:

- `fitforge-api/src/FitForge.Api/**`
- `fitforge-api/tests/**`

### Implementation

- [ ] T061 Add `src/FitForge.Api/Hosting/Retention/RetentionService.cs` — a `BackgroundService` running daily (D9). No package, no scheduler.
- [ ] T062 Add the retention window as one named constant, **30 days**, cited by both the purge and the UI copy that phase 9 renders (VI-027). If it changes, both change or neither does.
- [ ] T063 Permanently remove members whose `DeletedAtUtc` is more than the window past, together with every row they own, in one transaction per member.
- [ ] T064 Prune `SignInAttempt` rows older than their 15-minute window — the table is a counter, not a log, and keeping it is keeping personal data past its usefulness (invariant 10).
- [ ] T065 Log what was removed as counts only: never an email, never an address, never an internal `Id` (D12).

### Tests

- [ ] T066 **[D13-1]** `FitForge.Api.Tests` — enumerate every `FitForgeDbContext` entity type carrying a member reference and assert each is named in the purge. A later feature adding a member-owned table without extending the purge fails the gate rather than silently orphaning personal data (D9).
- [ ] T067 [P] `FitForge.Api.Tests` — a member soft-deleted 31 days ago is removed; one soft-deleted 29 days ago is not; one not deleted at all is not.
- [ ] T068 `FitForge.Api.Tests` — the purge removes the member's `Profile` and `Session` rows too, leaving no orphan.

**Gate (human-run)**: as phase 1.

---

## Phase 7: Sign in and register (US1, US2) — UI phase

**Territory**:

- `fitforge-web/src/**`
- `fitforge-web/.env.example`
- `fitforge-web/components.json`
- `fitforge-web/package.json`
- `fitforge-web/package-lock.json`

### Implementation

- [ ] T069 Generate the shadcn/ui primitives this feature needs — `input`, `label`, `select`, `card` — into `src/components/ui/`. Generated source, not a runtime dependency (plan §5).
- [ ] T070 Add `src/lib/session.ts` — read, write and clear `__Host-fitforge_session` with `HttpOnly`, `Secure`, `SameSite=Lax`, `Path=/`, no `Domain` (D4). `import "server-only"`.
- [ ] T071 Add the shared `Origin`-check helper and apply it to every mutating BFF route (D4). Three lines, one place.
- [ ] T072 [P] Add `src/app/api/bff/auth/sign-in/route.ts`, `register/route.ts`, `sign-out/route.ts` per `contracts/member.md` §6 — map, set or clear the cookie, and nothing else. **No generic pass-through route** (D11).
- [ ] T073 Map upstream unavailability to a service failure, never a credential failure (FR-017, `contracts/auth.md` §7). A member told their password is wrong when the server is down will change a password that was fine.
- [ ] T074 Build the sign-in screen at `src/app/(auth)/sign-in/page.tsx` to VI-001 through VI-016 — two-column grid at ≥1024px, left panel not rendered below it (VI-001), segmented control with Sign in selected (VI-007), field order email then password (VI-008), error above the button (VI-012), full-width 40px submit (VI-013).
- [ ] T075 Render the three declared deviations exactly as `spec.md` declares them: no "Forgot?" link (so VI-010's row is the label alone), and the register half of the segmented control switches the same card rather than routing away.
- [ ] T076 **Remove `FITFORGE_SESSION_SECRET` from `.env.example`** (D4). This design gives it no purpose, and a named secret nobody uses invites someone to make it load-bearing later without a decision.

### Tests and the loop

- [ ] T077 [P] Vitest — the BFF sign-in route maps 401 to a credential error and an unreachable API to a service error. Every row of the mapping table, as 001 did for health.
- [ ] T078 Vitest — the cookie is set with all five attributes, and its value is not readable from a non-`HttpOnly` path.
- [ ] T079 **Visual Compliance Loop** (`docs/sdlc/review-process.md`) against `screenshots/01-signin-desktop-{light,dark}.jpg`, in both themes, until the deviation table is empty or holds only the three `spec.md` declares.
- [ ] T080 Verify the responsive rules live at <1024px (VI-001, VI-017). No capture exists below 1024px — `spec.md` says so — so these are checked in a resized browser, not against an image.
- [ ] T081 **SC-003 by inspection**: after sign-in, `localStorage`, `sessionStorage` and every script-readable cookie hold nothing that authenticates, and the browser issues no request to the API origin. Recorded in this file with what was inspected.

**Gate (human-run)**: `npm run lint && npm run build && npm run typecheck && npm test` in
`fitforge-web`.

---

## Phase 8: Route protection (US2)

**Territory**:

- `fitforge-web/src/**`

### Implementation

- [ ] T082 Redirect an unauthenticated visitor from every authenticated route to sign-in, rendering **no** member data on the way (FR-007).
- [ ] T083 Render the sign-in route with **no app shell** — no sidebar, no bottom bar, no header (VI-015). This is a route-group decision, not a conditional inside the shell.
- [ ] T084 Add `GET /api/bff/me` and have the shell read the member from it server-side.
- [ ] T085 Ensure the back button after sign-out does not restore an authenticated view — the response carries no-store, and the shell re-reads the session on every navigation (US1 scenario 5).

### Tests

- [ ] T086 [P] Vitest — an unauthenticated request to an authenticated route redirects and renders no member data.
- [ ] T087 Vitest — the sign-in route renders no shell element.

**Gate (human-run)**: as phase 7.

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
