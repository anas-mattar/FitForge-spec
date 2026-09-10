# Contract — Member and Profile (BFF ⇄ API)

**Feature**: `002-identity-member-profile` | **Status**: proposed 2026-09-10
**Parties**: `fitforge-web` BFF route handlers ⇄ `fitforge-api`
**Constitution VII**: approved before either side implements against it.

Conventions from `contracts/auth.md` §0 apply unchanged. Every path here is authenticated
and every one is **`/me`-shaped**: there is no path, parameter, header or body field by
which a caller names *which* member. That absence is the enforcement of invariant 2 and
FR-008 — not a filter that could be forgotten, but a parameter that does not exist.

## 1. `GET /api/v1/me`

**200** — `<MemberSummary>` (auth §8) plus `profile`:

```json
{ "birthYear": 1990, "sex": "Female", "heightCm": 167.50 }
```

Every `profile` field is nullable; a member who has given nothing gets `null`s, not an
absent object. `heightCm` is centimetres, `DECIMAL(5,2)`, always canonical — the caller
converts for display and never sends back a converted value (invariant 4, FR-011).

**401** — `type: /problems/no-session`.

## 2. `PATCH /api/v1/me/preferences`

**Request** — every field optional; absent means unchanged. `null` is not accepted for any
of these four (they all have defaults and none is clearable).

| Field | Type | Values |
|---|---|---|
| `units` | string | `Metric` \| `Imperial` |
| `goal` | string | `Hypertrophy` \| `Strength` \| `Endurance` \| `GeneralFitness` |
| `experience` | string | `Beginner` \| `Intermediate` \| `Advanced` |
| `timeZone` | string | an IANA identifier the server can resolve |

**200** — the updated `<MemberSummary>`.

**422** — `type: /problems/validation`. An unresolvable time-zone identifier lands here
with `"timeZone": ["That time zone is not recognised."]` — the server, not the browser, is
the authority on what it can resolve (spec, Edge Cases).

`units` changes **nothing stored anywhere else**. It is a rendering instruction (FR-011,
invariant 4, VI-028). There is no code path in this feature that rewrites a persisted
measurement, and the test named in `plan.md` D8 is what keeps that true.

## 3. `POST /api/v1/me/password`

**Request**: `{ "currentPassword": string, "newPassword": string }`

**204** — the password is changed, and **every other** session for this member is revoked.
The presented session survives, so the member is not signed out of the tab they are using
(FR-013).

**401** — `type: /problems/invalid-credentials`, `title: "Your current password is
incorrect."`

Deliberately **distinguishable** from auth §3's message, and the difference is safe: the
caller has already proven they are this member. Telling them which field is wrong is a
usability gain that leaks nothing.

**422** — new password shorter than 10 characters, or identical to the current one.

## 4. `DELETE /api/v1/me`

**Request**: `{ "password": string }` — deletion re-authenticates. A stolen session should
not be able to destroy an account.

**204** — in one transaction:

1. `Member.IsDeleted = 1`, `DeletedAtUtc = now`, `DeletedBy = <member's PublicId>`;
2. every member-owned row is soft-deleted by the same marker;
3. **every** session for the member is revoked, including the presented one.

Subsequent sign-in returns auth §3's 401 — indistinguishable from a wrong password, so a
deleted account is not discoverable (FR-014).

**401** — wrong password; nothing is deleted.

Nothing is physically removed here. Physical removal happens only in §5, and never as a
rollback mechanism (`docs/sdlc/rollback-process.md`).

## 5. Retention (FR-015, invariant 10)

Not an endpoint. A hosted service in the API runs daily and **permanently** removes members
whose `DeletedAtUtc` is more than **30 days** ago, together with every row they own.

30 days is not a configurable convenience: `VI-027` states it to the member in the product,
so the number lives in one place and the two move together or neither does.

A soft delete alone does not satisfy invariant 10 — "MUST make it unrecoverable after the
stated retention window" is the half a soft delete cannot do, which is why this is in 002
and not deferred.

**The guard that keeps this true as the product grows**: a test enumerates every
`FitForgeDbContext` entity type carrying a member reference and asserts each is named in the
purge. A later feature that adds a member-owned table without extending the purge fails the
gate rather than silently orphaning personal data past its retention window.

## 6. What the BFF exposes to the browser

One route handler per screen need, never a proxy. The browser calls:

| Browser → BFF | BFF → API |
|---|---|
| `POST /bff/auth/sign-in` | auth §3, then sets the cookie |
| `POST /bff/auth/register` | auth §2, then sets the cookie |
| `POST /bff/auth/sign-out` | auth §4, then clears the cookie |
| `GET /bff/me` | §1 |
| `PATCH /bff/me/preferences` | §2 |
| `POST /bff/me/password` | §3 |
| `DELETE /bff/me` | §4, then clears the cookie |

**There is no generic pass-through route.** A `/bff/proxy/[...path]` would hand the browser
the whole API surface behind a cookie and make invariant 7 unenforceable, so the shape is
enumerated here and its absence is reviewable.

The BFF **maps** and nothing else: it holds no rule from
`modules/training/training-invariants.md`, opens no database connection, and never decides
what a member may do (invariant 7, ADR-001 §4.6).
