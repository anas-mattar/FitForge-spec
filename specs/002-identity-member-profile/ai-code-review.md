# AI Code Review — 002 Identity and Member Profile

**Reviewer**: three fresh-context agent sessions (Claude Opus 5), one per dimension — see Reviewer Provenance
**Date**: 2026-09-12
**Branches**: `fitforge` `002-identity-member-profile` (tip `398e75f`) · `fitforge-api` `002-identity-member-profile` (tip `2233d8e`) · `fitforge-web` `002-identity-member-profile` (tip `a3ca4b9`)
**Scope reviewed**: all nine implementation phases plus phase 11. API: `Features/Identity/`, `Features/Me/`, `Hosting/Authentication/`, `Hosting/Retention/`, `FitForge.Domain/Members/`, `Infrastructure/Persistence/` (configurations + both migrations), and `tests/`. Web: `src/app/api/bff/**`, `src/lib/` (session, bff, auth-transport, me-transport, api-auth, units), `src/app/(app)/**`, `src/app/(auth)/**`, `src/components/auth/`, `src/components/profile/`, `src/components/shell/`, `src/proxy.ts`, and `src/**/__tests__/`. Governance: `modules/training/training-invariants.md`, `spec.md`, `plan.md`, `data-model.md`, `contracts/auth.md`, `contracts/member.md`.
**Feature contract**: Critical lane. Ten training invariants bind. Contract-before-implementation (constitution VII) — `contracts/auth.md` and `contracts/member.md` were approved before phase 1. Three declared visual deviations, no others. No package outside `plan.md`.

## Reviewer Provenance

- **Reviewer**: fresh-context agent — claude-opus-5, three independent sessions (domain invariants · API identity security · web BFF and contracts)
- **Implementer**: claude-opus-5, the session that produced phases 1–9 and 11
- **Inputs provided**: the branch diffs in both code repositories, `spec.md`, `plan.md`, `data-model.md`, `contracts/`, and `modules/training/training-invariants.md`. Each reviewer was instructed that the feature's own documents — `tasks.md` above all — are **claims, not evidence**, and that agreement with them is not verification.
- **Attestation**: This reviewer did not produce the diff under review.

**Why three and not one.** Nine phases across three repositories is more than one reviewer reads carefully. The dimensions were assigned disjointly and the sessions could not see each other. That independence produced the review's strongest single signal: two reviewers, told nothing of one another, found F1 by different routes — one from invariant 7's enforcement split, one from an adversarial read of the throttle.

**What the owner verified personally**, rather than relaying: F1 (the missing header, end to end), F4 and F5 (`RequireAuthorization` on sign-out; `no-session` absent from the C# source), F7 (no `DateTimeKind`/JSON configuration anywhere in the API), F9 (sign-out appears in no component), F10 (no test file references the cookie module), and F12/F13 (the Origin comparison). Those are stated below as fact. The remainder are the reviewers' findings, reported with their evidence.

## Verdict

**REQUEST CHANGES** — 14 blocking findings. The feature's core property is genuinely well built: member isolation (invariant 2) is structural rather than conventional, and `MemberScopingTests.cs:141-213` plus the `/me` shape make it self-policing. What fails is the perimeter around it. Three defects make a shipped security control inert (F1 the throttle, F12/F13 the CSRF check), one P1 acceptance scenario has no implementation at all (F9, sign-out), two approved contracts are violated (F4, F5), and — the finding with the widest consequences — **five tasks are marked `[x]` in `tasks.md` against tests that grade a different module or do not exist** (F11). Residual risk concentrates not in the code but in the audit trail: the gates were run honestly and passed honestly, over a suite that does not assert what the record says it asserts.

## What was verified (evidence)

| Area | Evidence |
|---|---|
| Spec match (FRs implemented as specified) | FR-003/004/005/006/007/014 verified in code. **FR-016 fails** (F1, F2, F3, F6 — the throttle does not throttle in production). **US1 acceptance scenario 5 has no implementation** (F9 — no sign-out control exists in any component) |
| Visual-reference match | Phase 7 and 9 Visual Compliance Loop tables read against `screenshots/01-signin-desktop-{light,dark}.jpg` and `11-profile-desktop-{light,dark}.jpg`. Three declared deviations render as declared. **A fourth, undeclared, was found** (F14 — an added "Height:" row absent from the reference) |
| Feature contract held | No package outside `plan.md`; no unapproved table. Two migrations match `data-model.md` on every index, unique constraint and check constraint, with the enum ranges matching the C# enums exactly. One divergence: three declared column defaults are absent (N4) |
| Constitution / domain invariants | Item-by-item pass over all ten, below. Invariants 2 and 10 upheld; 4, 7 and 8 carry blocking findings; 1, 3, 5, 6 and 9 are not touched by this feature |
| Security (authn/authz, secrets, sensitive logging) | Token minting, storage and revocation verified correct — `RandomNumberGenerator.GetBytes(32)`, only `SHA-256` persisted, no read path back to the token. Password storage verified correct — PBKDF2-HMAC-SHA512, 210,000 iterations, rehash-on-verify real and persisted. Logging verified clean: five log statements across both repositories, none taking a password, token, email or identifier. **Throttling, CSRF and re-authentication all carry blocking findings** (F1, F2, F3, F6, F12, F13) |
| Scope guard | `scope-check-repos` PASS on all eleven phase commits; `git diff --stat` read for intent on each. No stray files. Phase 7's 20-vs-21 count is the rename reconciliation recorded in `tasks.md` |
| Rollback safety | Both migrations' down-paths verified clean and correctly ordered (child before parent, both `Restrict` FKs respected). Schema is additive. `rollback.md` matches what the migrations actually do |

## The ten training invariants, item by item (SC-006, critical-delivery item 2)

Every verdict below was reached by opening the code, not by reading `data-model.md`'s trace.

| # | Invariant | Touched | Upheld by | Verdict |
|---|---|---|---|---|
| 1 | Completed training is immutable | No | — | **N/A** — no `WorkoutSession`, `SetEntry` or `SessionAdjustment` type exists on this branch. The only entities added are `Member`, `Profile`, `Session`, `SignInAttempt` |
| 2 | Every record belongs to one member, and never leaves them | Yes | `MeEndpoints.cs:34` (a group with no route parameter), `:55`, `:88`, `:174`, `:263` — every query keyed on `current.Member.Id`; `CurrentMember.cs:38` the sole identity source; `ProfileConfiguration.cs:28-36` restrict FK + unique `MemberId`; `MemberScopingTests.cs:64,108,142,216` | **UPHELD** — structurally, not by convention. The best-built property in the feature |
| 3 | Master data is soft-deleted | No | — | **N/A** — `Exercise`, `Gear`, `MuscleGroup`, `Program`, `ProgramDay` do not exist yet. Retention physically deletes only `Member`/`Profile`/`Session`, none of which this invariant enumerates |
| 4 | Canonical units and time, converted only at the edge | Yes | Storage half upheld: `Profile.cs:47` + `ProfileConfiguration.cs:40` (`decimal(5,2)` cm, never float); instants `datetime2(3)` written through the injected clock; `MeEndpointTests.cs:54` proves a units change leaves the column byte-identical | **F7 (BLOCKING)** — the edge half is broken: a stored UTC instant crosses the wire without a UTC marker and is re-read as local time |
| 5 | Derived numbers are never stored as truth | No | — | **N/A** — nothing derived is persisted. `memberSince` is `CreatedAtUtc` read directly; `expiresAtUtc` is read back from the stored row rather than recomputed |
| 6 | A set is physically possible | No | — | **N/A** — no `SetEntry`, `Reps`, `LoadKg` or `Rpe` anywhere on the branch |
| 7 | Domain rules live in the API, never in the BFF or the browser | Yes | Negative half upheld: no DB driver in `fitforge-web/package.json`; `session.ts` and `api-auth.ts` carry `server-only`; `me-transport.ts:25-52` forwards bodies verbatim; `PasswordPolicy` exists only in `FitForge.Domain` and is not mirrored in TS; `SignInCard.tsx:82` uses `noValidate` and checks no policy | **F1 (BLOCKING)** — the BFF's *positive* obligation under the split (supply the source address) is unimplemented. **F8 (NON-BLOCKING)** — one constitutional constant is duplicated in the browser with nothing keeping the two in step |
| 8 | Auditability and identity | Yes | `BusinessEntity.cs:32,42,44-64`; `MemberConfiguration.cs:26-29,48-54`; `ProfileConfiguration.cs:19-22,42-48`; internal `Id` never serialized; `BearerSessionHandler.cs:66-67` puts `PublicId` in the claim; retention logs counts only | **F8-GOV (BLOCKING, needs a decision)** — two of the four persisted entities carry neither the PK standard nor the audit fields, waived against a rulebook rather than by amendment. **N2** — `CreatedBy` records `"self"`, which is not a value the contract defines |
| 9 | One authoritative state per member | No | — | **N/A** — no `WorkoutSession` or `Program` exists. Multiple concurrent auth `Session` rows are intentional and are a different concept from this invariant's *active session* |
| 10 | Personal data is minimal and deletable | Yes | Minimal: `Profile.cs:36,38,47` collects birth *year*, sex and height only. Deletable: `MeEndpoints.cs:279-303` transactional soft delete + revoke-all; `RetentionPolicy.cs:25,42-47`; `RetentionRunner.cs:40-83`; wired at `Program.cs:40-41`; `RetentionTests.cs:78,111,131,146,161,179,194` | **UPHELD.** `RetentionTests.cs:78` — asserting that every entity with an FK to `Member` is named in the purge — is the best forward-looking enforcement in the feature. Two adjacent findings: **F15** (the promised recovery does not exist) and **N3** (the purge set is decorative) |

## Findings

### F1 — The sign-in throttle's per-source bucket is one global counter — BLOCKING

Found independently by two reviewers, and verified by the owner end to end.

`AuthEndpoints.cs:168` reads the source address from `X-Forwarded-For`. `contracts/auth.md:91` specifies the bucket as "per source address, **supplied by the BFF as `X-Forwarded-For`**", and `SignInThrottle.cs:30-32` restates it: *"The BFF forwards the caller's address; it does not decide what to do about it."*

The BFF never does. `auth-transport.ts:43-52` sets exactly `content-type` and an optional `authorization`; a search of the whole `fitforge-web/src` tree for `forwarded` returns one comment and one type name. So the header is absent on every production request, `source` is `""`, and `SignInThrottle` hashes `salt + "|" + ""` into **one constant `SourceHash` shared by every sign-in attempt in the system**.

*Failure scenario.* An attacker sends 30 sign-ins with a garbage password to any address. Attempt 31 — a different member, correct password, own laptop — matches `oldestSourceAttempt` and receives **429**. Every member is locked out for up to 15 minutes, renewable indefinitely at 2 requests/minute. `ClearEmailAsync` does not clear the source bucket, so a success does not relieve it. A per-source defence has become a whole-product denial of service costing 30 requests.

The suite is green because `CredentialEndpointTests.cs:42-46` adds the header itself — the throttle tests exercise a header the production client never sends.

Second half, for whoever fixes it: once the BFF does send it, the API takes it raw. `Program.cs` configures no `UseForwardedHeaders` and no known-proxy list, so any caller reaching the API directly gets a fresh bucket per request by varying one header.

*Action: the BFF forwards the address, and the API validates it against a known-proxy configuration. A test must assert the header leaves the BFF — the current tests fabricate it.*

### F2 — The throttle is check-then-act across a ~200 ms window — BLOCKING

`AuthEndpoints.cs:172` checks; `:193`/`:201` records — with the deliberate 210,000-iteration PBKDF2 between them, in two independent read-committed statements, with no transaction, lock or unique constraint serializing them.

*Failure scenario.* 500 parallel sign-ins for one address: all 500 read zero rows in the window, all 500 proceed to full verification. 500 unthrottled password guesses, plus roughly 100 CPU-seconds of PBKDF2 from one burst. FR-016 holds only for strictly serialized attempts, which an attacker has no reason to use.

*Action: serialize the check and the record, or move the decision to a single atomic statement.*

### F3 — `POST /auth/register` is unthrottled and is an email-existence oracle — BLOCKING

`AuthEndpoints.cs:53-59` takes no `SignInThrottle` and calls none; no rate-limiting middleware exists. `contracts/auth.md:49` lists **429** under register; nothing implements it.

*Failure scenario (a).* Unauthenticated, unlimited: a 409 means the address exists, a 201 means it did not. The decoy hash and the "count addresses that do not exist" rule exist to close exactly this oracle on `/sign-in`; `/register` hands it back. The contract's rationale for tolerating the asymmetry assumed the 429 on line 49, which was never built.
*Failure scenario (b).* Each call reaches a full 210,000-iteration hash at `:103`, unauthenticated. ~50 concurrent requests saturate the instance.

*Action: throttle register on the same buckets as sign-in.*

### F4 — Sign-out is not idempotent, and both the contract and a code comment say it is — BLOCKING

Verified by the owner. `AuthEndpoints.cs:34` carries `.RequireAuthorization()`. `BearerSessionHandler` → `SessionService.ResolveAsync` filters `RevokedAtUtc == null`, so the second call is rejected by the pipeline and the handler never runs. The handler's own comment at `:239-240` claims "Idempotent by construction"; `contracts/auth.md:69-71` promises 204 for an already-revoked token.

*Failure scenario.* The BFF retries a sign-out after a network blip, or the member double-clicks: first call 204, second **401**.

`CredentialEndpointTests.cs:332-335` asserts the 401 and rationalizes it in a comment. A test that documents a divergence from an approved contract is not a test that closes it.

*Action: decide which document is wrong and change that one. If the contract stands, sign-out must not require authorization.*

### F5 — `/problems/no-session` is specified twice and implemented nowhere — BLOCKING

Verified by the owner: a search of every `.cs` file under `fitforge-api/src` for `no-session` returns **zero** matches. `contracts/auth.md:78` and `contracts/member.md:24` both specify it. Every 401 is produced by the authorization pipeline and bodied by the framework default, so the `type` discriminator the contracts told consumers to branch on never appears.

*Failure scenario.* Any consumer written against the contract — a later mobile client, a hardened BFF, an integration test — cannot distinguish an expired session from invalid credentials at the discriminator it was told to use. No test asserts the `type` on any 401.

*Action: emit the specified problem type, or amend both contracts with a recorded approver.*

### F6 — Change-password and delete-account re-authentication is unthrottled — BLOCKING

`MeEndpoints.cs:178` and `:268` each perform a full password verification. Neither takes a throttle, consults one, or records an attempt.

*Failure scenario.* An attacker holding a session token (a shared machine, a proxy log) guesses `currentPassword` at request rate forever; 401 vs 422 vs 204 is a clean oracle, and the per-email bucket never sees it. `MeEndpoints.cs:265-267` gives the justification — "A stolen session should not be able to destroy an account" — which an unlimited guessing loop against `DELETE /api/v1/me` defeats. This is the one action with no undo after the retention window.

*Action: route both re-authentications through the throttle.*

### F7 — UTC instants reach the browser unmarked and render as the wrong date — BLOCKING

Verified by the owner: searching the entire API source for `DateTimeKind`, `SpecifyKind`, `ConfigureHttpJsonOptions` and `JsonSerializerOptions` returns **zero** matches, and `Program.cs` configures no JSON options. EF Core's SQL Server provider returns `datetime2` as `DateTime` with `Kind = Unspecified`, and `System.Text.Json` serializes that **without** a `Z`. `AccountCard.tsx:234` then calls `new Date(instant)`, which ECMAScript parses as **local** time for an offset-less date-time, and formats it with `timeZone: "UTC"`.

*Failure scenario.* An account created at `2026-09-10T03:00:00Z`, viewed from Kuala Lumpur (UTC+8, the first entry in `PreferencesCard.tsx:31`), renders **"Member since 9 Sep 2026"** — a day early. West of UTC it renders a day late. The same account shows different join dates on the member's phone and laptop. No test asserts the serialized shape; `data-model.md:167` records invariant 4 as satisfied.

*Action: make the API emit an offset, and assert the wire shape in a test.*

### F8-GOV — Two persisted entities were exempted from invariant 8 by the wrong instrument — BLOCKING, needs an owner decision

`Session` and `SignInAttempt` carry no `PublicId`, no `CreatedBy`, no `UpdatedAtUtc`/`UpdatedBy` — confirmed in the migration, not only the model. Invariant 8 requires them of "**every persisted entity**".

The problem is not the omission; it is what waived it. `plan.md:189-193` and `data-model.md:105` frame the deviations against `database-rules.md` — a rulebook, a *lower rung* — and argue them on that document's narrower term *business entity*. `plan.md:61-63` then asserts that no invariant is violated, and `data-model.md:171` records invariant 8 as satisfied. The invariants file forecloses the route in its own preamble: *"A plan that violates one stops and is reported; it is never quietly justified… this one changes only by constitutional amendment."* The reviewer checked: `training-invariants.md` has exactly one commit and has **never been amended**.

*Failure scenario — why this is not paperwork.* Every future feature now has a worked precedent: call an entity "not a business entity" in `data-model.md`, cite the rulebook, ship without `PublicId` or `CreatedBy`. The next entity down that path is one where it matters — a `SessionAdjustment`, which invariant 1 requires to be *itself auditable*. A correction that rewrote a member's training history would then have no recorded author, and the only fix is a migration against live data.

*Action: owner decides — add the columns, or amend `training-invariants.md` to name the exception with a recorded approver under constitution 1.1.0. The engineering argument in `Session.cs:12-22` is sound; the instrument was not. This is the one finding no agent should close.*

### F9 — The product has no way to sign out — BLOCKING

Verified by the owner: `sign-out` appears in exactly five files — the BFF route, two transports, `proxy.ts`, and a transport test. **No component references it.** Not `Header.tsx`, not `navigation.ts`, not `Sidebar.tsx`, not `AccountCard.tsx`.

*Failure scenario.* A member signs in on a shared laptop and wants to sign out. There is no control, at any breakpoint. The only way to end a session is to delete the account.

`spec.md:67` (US1 acceptance scenario 5) requires it. `spec.md:230-244` declares three deviations, none of them this. And `tasks.md:511` marks **T085 `[x]`** — "the back button after sign-out does not restore an authenticated view" — a task guarding a path no member can take. The reference screenshot has no sign-out control either, so this was never designed in; the spec still requires it.

*Action: build it, or declare the deviation and amend the spec's acceptance scenario with a recorded approver.*

### F10 — T078 is marked `[x]` and the test it names does not exist — BLOCKING

Verified by the owner: a search for `__Host-`, `httpOnly`, `sameSite`, `writeSessionToken`, `clearSessionToken` and `SESSION_COOKIE` across `fitforge-web/src` returns `session.ts` and five route handlers — **no test file**. `src/lib/session.ts` has no test of any kind.

*Failure scenario.* Someone flips `httpOnly: true` to `false`, or drops the `__Host-` prefix. Lint passes, build passes, `npm test` passes, the gate exits 0, and the session token becomes readable from `document.cookie`. The only evidence FR-005 and SC-003 ever held is the by-hand inspection at `tasks.md:461-486` — which the document itself flags as partly synthetic.

*Action: write the test T078 claims. Until then SC-003 rests on one manual inspection that no longer has a witness.*

### F11 — The audit trail asserts work that does not exist — BLOCKING (the pattern, not one instance)

F9 and F10 are not isolated. Five tasks are marked `[x]` against tests that grade a different module or do not exist:

| Task | Claims | Actually |
|---|---|---|
| T077 | "the **BFF sign-in route** maps 401 to a credential error and an unreachable API to a service error" | `api-auth.test.ts:22-25` imports `postSignIn` from `../auth-transport` — it grades the *transport*. No test file imports anything under `src/app/api/bff/` |
| T078 | a Vitest covering the cookie's five attributes | No such test exists (F10) |
| T084 | "Add `GET /api/bff/me` **and have the shell read the member from it server-side**" | The shell calls `getMe()` directly (`(app)/layout.tsx:27`). The route has no caller and is dead code |
| T085 | the back button after sign-out | No sign-out exists (F9) |
| T096 | the loop ran "until the deviation table… holds only the declared deviations" | A fourth, undeclared deviation ships (F14) |

*Failure scenario.* Change `sign-in/route.ts:32-34` so an unreachable API is reported as a credential failure — the precise outcome `contracts/auth.md:109-111` says MUST NOT happen, because a member told their password is wrong will change a password that was fine. All 76 tests still pass. Delete the origin check from `me/preferences/route.ts:8` and the suite is still green.

This is the finding with the widest blast radius, and it is a governance finding as much as a technical one: every gate in this feature was run honestly by a human and passed honestly, over a suite that does not assert what the record says it asserts. Nine phases of green gates certify less than the file claims.

*Action: correct the five task records to say what was actually built, then write the missing tests as a remediation phase. The record must be corrected even if the tests are deferred — a false `[x]` is worse than an open box.*

### F12 — The Origin check compares host only, never scheme — BLOCKING

Verified by the owner. `bff.ts:41`: `return new URL(origin).host === host;`. `new URL("http://app.fitforge.example").host` equals the `Host` header of the HTTPS request — both omit their default ports.

*Failure scenario.* A network attacker on hostile Wi-Fi answers port 80 for the site's host and serves a page that fetches the BFF with `credentials: "include"`. `Origin: http://…` passes the check. `SameSite=Lax` treats it as same-site, so the `__Host-` cookie is attached. Belt and suspenders fail together. `bff.test.ts:42-50` tests the *port* dimension and never the scheme — the near-miss makes the gap look covered.

*Action: compare the full origin, scheme included.*

### F13 — The missing-Origin allowance rests on a claim the codebase contradicts — BLOCKING

Verified by the owner: `bff.ts:23-29` returns `true` when `Origin` is absent, justified by "the server-rendered paths that call it during a render." No server-rendered path in this codebase fetches any `/api/bff/*` URL — `(app)/layout.tsx:27` and `profile/page.tsx:31` call `getMe()` directly.

The only permissive branch in the CSRF check is therefore unreviewed rather than reasoned: a non-browser caller sends no `Origin` and is admitted.

*Action: return `false` for mutating routes, leaving SameSite as the fallback rather than the only defence.*

### F14 — An undeclared visual deviation, and it is inert — BLOCKING

`PreferencesCard.tsx:113-117` inserts a "Height: —" row between Units and Goal. The reference screenshots and `spec.md:206-210` (VI-019 → VI-020) have no such row; it shifts every control below it. `spec.md:230-244` declares three deviations and this is a fourth — an *addition*, not an omission — absent from the phase 9 deviation table.

It is also permanently empty: no endpoint in `contracts/member.md` and no UI anywhere writes `birthYear`, `sex` or `heightCm`, so `heightCm` is `null` for every real member. The `167.5 cm → 5' 6"` evidence at `tasks.md:616` came from the stub API, which means VI-028 ("units re-render every displayed value") was demonstrated by a value no member can have.

*Action: remove the row, or declare the deviation and give the field a way to be set.*

## Non-blocking findings

Recorded in full so a remediation phase can pick them up; none blocks merge on its own.

| # | Where | What |
|---|---|---|
| N1 | `SignInThrottle.cs:69-79,129-137` | `Retry-After` is computed from the 10th-oldest attempt, not the oldest — over-reports by up to a full window, under-reports under spraying |
| N2 | `AuthEndpoints.cs:106,115` | `CreatedBy = "self"`; the contract defines a `PublicId` or the literal `system`. The update paths get it right, so the inconsistency is within one feature |
| N3 | `RetentionRunner.cs:58-70` | `RetentionPolicy.Purged` is never read — the runner hardcodes three deletes. The gate fails on the *name* being absent from the set, not on the data surviving |
| N4 | `20260910140536_AddMemberAndProfile.cs:24,26,27` | Three column defaults declared in `data-model.md:38,40,41` are absent from the migration |
| N5 | `AuthEndpoints.cs:69` | Email validation accepts `@` — `POST /register` with `{"email":"@"}` returns 201 |
| N6 | `AuthEndpoints.cs:191,197` | `PasswordPolicy.MaximumLength` is enforced on register and change-password but not on sign-in; a 10 MB password is hashed |
| N7 | `SessionService.cs:102-107` | No absolute session lifetime — a token used weekly never expires. A design gap rather than a divergence, but it should be a recorded decision on a Critical feature |
| N8 | `IdentityServiceCollectionExtensions.cs:49-52` | The decoy hash is computed on first use, not at startup as two comments claim — the first sign-in after a deploy is measurably slower, at the one instant an attacker can trigger |
| N9 | `CredentialEndpointTests.cs:34-48` | Most test sign-ins omit the source header, so the suite shares one bucket (F1's) — the suite is flaky by construction once 30 failures accumulate in a run |
| N10 | `CredentialEndpointTests.cs:283-311` | `A_successful_sign_in_does_not_clear_the_source_bucket` signs in as an address with zero failure rows; it would pass against an implementation that deleted the whole table |
| N11 | `MeEndpoints.cs:321-323` | `PATCH /me/preferences` with a non-object body (`[]`, `123`, `{}`) returns 200 and writes an audit stamp claiming the member changed their preferences |
| N12 | `MeEndpoints.cs:219-226` | The change-password compare-and-swap compares base64 under the server collation, which is case-insensitive by default — the same argument `data-model.md:44-50` makes for `NormalizedEmail`, not applied here |
| N13 | `AuthEndpoints.cs:210` | The one write in the feature that bypasses the injected `TimeProvider` |
| N14 | `AuthEndpoints.cs:144` | `Location` points at `/api/v1/members/{PublicId}`, a route that is not mapped |
| N15 | `units.ts:43` | `heightCm === null` misses `undefined` → renders `NaN`. `== null` fixes it |
| N16 | `AccountCard.tsx:107-111,154` | The "Password changed." confirmation is dead code — `onDone()` unmounts the form first, so a successful change is silent |
| N17 | `PreferencesCard.tsx:68-71` | A rejected save leaves the UI showing the value the server refused |
| N18 | `bff.test.ts:71` | A self-referential assertion that computes its expectation from the object under test |
| N19 | `me-transport.ts:52` + three routes | A `application/problem+json` body is re-labelled `application/json` |
| N20 | `auth-transport.ts:72-81` | `Retry-After` is dropped on 429, so the sign-in screen can never say how long to wait |
| N21 | `proxy.ts:24` | `no-store` does not govern Next's client Router Cache, which is what serves a Back after `router.push()` |
| N22 | `AccountCard.tsx:93,176`, `PreferencesCard.tsx:56` | No busy guard on any profile mutation — double-click sends two DELETEs |
| N23 | `(app)/layout.tsx:11-14` | The comment claims the redirect is thrown before any child renders; in the App Router layouts and pages render concurrently, which is why `profile/page.tsx` needed its own guard. The code is safe; the stated invariant is wrong, and `(app)/page.tsx` has no second guard |
| N24 | `AccountCard.tsx:67` + `RetentionPolicy.cs:19-25` | The 30-day window is hard-coded in the browser and rendered to the member as a promise, with nothing tying it to the C# constant |
| N25 | `MeEndpoints.cs:281`, `MemberConfiguration.cs:34-36` | Nothing ever un-deletes a member, yet the UI promises "recoverable for 30 days"; the unfiltered unique index also holds the address for those 30 days, so the member is told both "no such account" and "already registered" |

## Constitution re-check (post-implementation)

**FAIL** — two principles engaged and not satisfied.

- **Constitution VII (contract before implementation)**: PASS at plan time, **FAIL as built**. Both contracts were approved before phase 1 and the implementation diverges from them in three places (F4 sign-out idempotence, F5 the problem type, F1's `X-Forwarded-For` obligation, F3's 429). A contract approved and then not implemented is the failure mode VII exists to prevent.
- **Domain invariants (constitutional force)**: **FAIL** — invariant 4 (F7), invariant 7 (F1), invariant 8 (F8-GOV). F8-GOV is the structural one: a plan waived a constitutional invariant by citing a rulebook.
- **Constitution I, amendment authority**: PASS. All four phase-1 amendment requests carry an approver line, A3 was correctly withdrawn rather than retro-approved, and the scope check caught the one retroactive widening attempt.
- **Constitution IV (architecture)**: PASS. No unapproved package, no unapproved pattern. The BFF/API split holds in the direction that was tested.

## Test coverage observed

**`fitforge-api`** — 128 tests. Genuinely strong where it is strong: `MemberScopingTests.cs:141-213` asserts member isolation *structurally* (no endpoint under `/me` may declare a way to name a member) with a behavioural backstop, and the mutation test recorded at `tasks.md:271` was really performed. `RetentionTests.cs:78` asserts that every entity with an FK to `Member` appears in the purge set — forward-looking enforcement that will fail for a future developer, which is the hardest kind to write. `MemberPasswordHasherTests.cs:129-180` counts hash invocations rather than timing them, which is the right way to test a timing defence. Weaknesses: N9 (the suite shares F1's global bucket and is flaky by construction), N10 (one test asserts nothing).

**`fitforge-web`** — 76 tests across 9 files, 5 of them this feature's. What they actually assert: `originIsAllowed` in isolation (7 cases), transport mapping (7 cases), the `(app)` layout's four redirect branches, `formatHeight` (6 cases), and several `readFileSync` + `toContain` source-text greps. **No test imports any file under `src/app/api/bff/`, and `src/lib/session.ts` has no test at all.** Untested: all six BFF route handlers; the cookie's five attributes; sign-out's revoke-before-clear and its cookie-clearing on API failure; `getMe`'s failure mapping; the rule that a failed delete must NOT clear the cookie; `proxy.ts` entirely; and every component's error-state behaviour. The structural greps are brittle in a specific way — rename a function and four assertions silently stop matching anything except a `toBeGreaterThan(0)` guard.

The asymmetry is the story: the API suite tests behaviour the documents claim, and the web suite tests a layer below the one the documents name.

## Residual risk

**Where it concentrates: the audit trail, not the code.** Eleven phase gates were run by a human and exited 0, honestly. F11 shows what those gates were grading. The single most security-critical artifact in the feature — the session cookie's attributes — is asserted by nothing, and a task record says it is. The correct reading of this feature's green history is "the build compiles and the tests that exist pass", not "the behaviour in `tasks.md` is verified".

**What must happen before merge**: F1, F9, F10, F11 and F12 at minimum. F1 because a shipped product would be one attacker and 30 requests from a full sign-in outage; F9 because a P1 acceptance scenario has no implementation; F10 and F11 because the record is wrong, and a wrong record is what stops the next reviewer looking; F12 because the CSRF defence and its fallback fail to the same attack.

**What is genuinely safe to carry**: member isolation. Invariant 2 is upheld structurally, and if only one property of an identity feature were to survive review intact, that is the right one. Both the `/me` shape and `RetentionTests.cs:78` are patterns the rest of the feature should have been held to — which is precisely why N3 and N24 stand out: the same authors knew how to make an invariant enforce itself, and did not do it twice.

**What needs the owner rather than a patch**: F8-GOV. Add the columns or amend the invariant — but a plan cannot waive a rule of constitutional force, and the precedent is worth more than the two entities it was used on.
