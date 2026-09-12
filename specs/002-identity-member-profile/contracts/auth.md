# Contract — Authentication (BFF ⇄ API)

**Feature**: `002-identity-member-profile` | **Status**: proposed 2026-09-10
**Parties**: `fitforge-web` BFF route handlers (client) ⇄ `fitforge-api` (server)
**Constitution VII**: this contract is approved before either side implements against it.

The browser is **not** a party to this contract. It never sees these paths, these payloads,
or the session token (invariant 7, FR-005, annotation A6). Everything here happens
server-to-server.

## 0. Conventions

- Base path `/api/v1`. All bodies `application/json`; all errors `application/problem+json`
  (RFC 9457, per ADR-001 §4.4).
- Every instant is UTC ISO-8601 with `Z` (invariant 4).
- No response on any path carries an internal `Id`; `PublicId` only (invariant 8).
- The API never sets a cookie. It returns a token; **the BFF owns the cookie.**
- The API trusts no caller-supplied member identity. On every authenticated path the member
  is resolved from the session token and from nothing else (FR-008, invariant 2).

## 1. Session token

An opaque, uniformly random 256-bit value, base64url-encoded without padding (43 chars).

The API stores **only** `SHA-256(token)`. A database read therefore yields no usable
session. Lookup is by hash; there is no reverse path.

Sent by the BFF as `Authorization: Bearer <token>`.

## 2. `POST /api/v1/auth/register`

**Request**

| Field | Type | Rules |
|---|---|---|
| `email` | string | trimmed, ≤ 254 chars, must parse as an address |
| `password` | string | ≥ 10 chars, ≤ 256 chars, no composition rules (FR-002) |
| `displayName` | string | trimmed, 1–60 chars |

**201** — `{ "token": "<43 chars>", "expiresAtUtc": "…", "member": <MemberSummary> }`

**409** — `type: /problems/email-taken`, `title: "That email is already registered."`
Matching is case-insensitive and post-trim (FR-001). Registration is **not** an oracle worth
protecting here: the product has no cross-member surface, and a registration form that lies
about a taken address cannot complete. Recorded as a deliberate asymmetry with §3.

**422** — `type: /problems/validation`, with `errors: { "<field>": ["<message>"] }`.

**429** — see §6.

## 3. `POST /api/v1/auth/sign-in`

**Request**: `{ "email": string, "password": string }`

**200** — `{ "token": "<43 chars>", "expiresAtUtc": "…", "member": <MemberSummary> }`

**401** — `type: /problems/invalid-credentials`,
`title: "Email or password is incorrect."` (VI-012, verbatim).

This response is returned for **all** of: unknown email, wrong password, soft-deleted
member. Identical body, identical status, and the server performs a full password
verification against a fixed decoy hash when no member is found, so the three cases are not
separable by timing either (FR-004).

**429** — see §6. **503** — see §7.

## 4. `POST /api/v1/auth/sign-out`

Authenticated. Revokes the presented session server-side (`RevokedAtUtc` set), then **204**.
Idempotent: an already-revoked or unknown token is also 204 — the caller learns nothing, and
the BFF clears its cookie either way (FR-006).

## 5. `GET /api/v1/auth/session`

Authenticated. Resolves the presented token.

**200** — `{ "member": <MemberSummary>, "expiresAtUtc": "…" }`
**401** — `type: /problems/no-session`. Returned for expired, revoked, unknown, and
"belongs to a soft-deleted member" alike.

Sliding expiry: a successful resolve extends `ExpiresAtUtc` to now + 14 days, at most once
per hour (so the hot path is not a write per request).

## 6. Throttling (FR-016)

Two fixed windows of 15 minutes:

| Bucket | Limit |
|---|---|
| per submitted email (normalized, whether or not it exists) | 10 |
| per source address, supplied by the BFF as `X-Forwarded-For` | 30 |

Sign-in (§3) and the `/me` re-authentications count **failed** attempts. Registration (§2)
counts **every** attempt, whatever its outcome — see the amendment below.

Exceeding either returns **429** `type: /problems/too-many-attempts` with `Retry-After` in
seconds.

The email bucket counts attempts against **addresses that do not exist** exactly as it counts
attempts against ones that do. This is what stops 429-vs-401 from becoming the existence
oracle §3 spent a decoy hash to close.

A successful sign-in clears that email's bucket. The source bucket is not cleared — one
success does not license thirty more guesses.

### Amendment — a successful registration is not cleared

**Amended 2026-09-12. Approved by**: anas.m. **Reason**: feature 002 review finding F3,
second failure scenario.

This section originally counted failed attempts only, and a registration that succeeds has
not failed at anything. Clearing it turned out to uncap the endpoint entirely: the clearing
deletes every row carrying the address, **including the one the attempt itself wrote a
moment earlier**, so a script registering distinct new addresses returns the source count to
zero after every success and each call still reaches the deliberate 210,000-iteration hash,
unauthenticated and without limit.

So a source may register **30 times in 15 minutes**; the thirty-first is refused, whether or
not any of the thirty failed. A successful registration is recorded and is not cleared.

**The cost, accepted knowingly**: thirty genuine signups from one office, gym or NAT inside
fifteen minutes will throttle the thirty-first. The remedy is operational — trust that site
as its own entry in `Security:TrustedProxies` so it gets its own bucket, or raise the limit —
not a return to clearing, which cannot cap a success at all.

One side effect is accepted rather than compensated: a successful registration also occupies
one slot in its own **email** bucket for the rest of the window, leaving the new member nine
first sign-in attempts rather than ten. Exempting it would need a second row type or an
outcome column, which is a schema change and a far larger thing than the defect.

That side effect is observable, so it is stated rather than left to be found: **for fifteen
minutes after signup, and only then, an address runs out of sign-in attempts one earlier than
an address that does not exist.** Everywhere else the two are indistinguishable, which is the
property §3's decoy hash exists to hold. The residue leaks "this address registered in the
last fifteen minutes" at a cost of ten requests, where §2 already answers the larger question
"does this address exist" in one — so it is dominated by an oracle this contract accepts
openly, not a new one.

## 7. Upstream unavailability (FR-017)

If the API is unreachable, times out, or answers 5xx, the BFF returns to the page a
**service** failure, never a credential failure: `type: /problems/service-unavailable`,
`title: "FitForge is unavailable right now. Please try again."`

The sign-in screen MUST NOT render "Email or password is incorrect." for this case. A member
told their password is wrong when the server is down will change a password that was fine.

## 8. `MemberSummary`

```json
{
  "publicId": "0199…",
  "email": "member@example.com",
  "displayName": "Anas",
  "memberSince": "2026-08-19T00:00:00Z",
  "units": "Metric",
  "timeZone": "Asia/Kuala_Lumpur",
  "goal": "Hypertrophy",
  "experience": "Intermediate"
}
```

`memberSince` is `CreatedAtUtc`; the profile screen renders it as `19 Aug 2026` (VI-024).
Formatting is the browser's job — the API emits the instant (invariant 4).
