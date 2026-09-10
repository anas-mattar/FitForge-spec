# Data Model: Identity and Member Profile

**Feature**: `002-identity-member-profile` | **Date**: 2026-09-10
**Governing rules**: `docs/rulebooks/database-rules.md` (schema standards) and
`modules/training/training-invariants.md` §2, §4, §8, §10 (constitutional force).

FitForge's first four tables. Everything the product stores afterwards hangs off `Member`,
so the choices here are the ones that are expensive to change later.

## Shared standards, applied

Every business entity below carries, per `database-rules.md`:

- `Id BIGINT IDENTITY(1,1) PRIMARY KEY` — internal, never exposed (invariant 8)
- `PublicId UNIQUEIDENTIFIER NOT NULL` with `UQ_<Table>_PublicId` — the only identifier
  that may appear in a URL, payload, or log
- `CreatedAtUtc DATETIME2(3) NOT NULL`, `CreatedBy NVARCHAR(64) NOT NULL`,
  `UpdatedAtUtc DATETIME2(3) NULL`, `UpdatedBy NVARCHAR(64) NULL`
- soft delete where the entity has one: `IsDeleted BIT NOT NULL DEFAULT 0`,
  `DeletedAtUtc DATETIME2(3) NULL`, `DeletedBy NVARCHAR(64) NULL`

`PublicId` is generated with `Guid.CreateVersion7()` — time-ordered, so its unique index
does not fragment the way v4 does, while remaining opaque and non-enumerable.

`CreatedBy` / `UpdatedBy` hold a member's `PublicId` as text, or the literal `system` for
rows written by the retention service. They never hold an internal `Id`.

## Member

The login identity and the account. One row per person.

| Column | Type | Notes |
|---|---|---|
| `Email` | `NVARCHAR(254) NOT NULL` | as the member typed it, trimmed — for display |
| `NormalizedEmail` | `NVARCHAR(254) NOT NULL` | `UQ_Member_NormalizedEmail`; see below |
| `PasswordHash` | `NVARCHAR(256) NOT NULL` | PBKDF2 composite, format in `plan.md` D2 |
| `DisplayName` | `NVARCHAR(60) NOT NULL` | |
| `Units` | `TINYINT NOT NULL DEFAULT 0` | `0 Metric`, `1 Imperial` — display only |
| `TimeZone` | `NVARCHAR(64) NOT NULL DEFAULT 'UTC'` | IANA identifier (FR-012) |
| `Goal` | `TINYINT NOT NULL DEFAULT 0` | `0 Hypertrophy` … `3 GeneralFitness` (VI-020 order) |
| `Experience` | `TINYINT NOT NULL DEFAULT 1` | `0 Beginner`, `1 Intermediate`, `2 Advanced` |
| *(soft delete + audit as above)* | | |

**`NormalizedEmail` exists as a column, not as an index expression.** FR-001 requires
uniqueness that is case-insensitive and post-trim. SQL Server's default collation would give
case-insensitivity by accident, on a setting a future DBA can change under us — and it would
not give trimming at all. Normalizing explicitly (trim, then `ToUpperInvariant`) makes the
rule a property of our schema rather than of the server's configuration, and
`UQ_Member_NormalizedEmail` then enforces it in the one place application code cannot forget
(`database-rules.md`, "Constraints Mirror Invariants").

`ToUpperInvariant` rather than `ToLowerInvariant`: lowercasing has locale traps (the Turkish
dotless i) that uppercasing does not, and ASP.NET Core Identity makes the same choice for
the same reason.

**Enums are stored as `TINYINT`.** The wire contract uses names, so the number is never
member-visible and renaming a member-facing label costs no migration. A check constraint
bounds each column to its defined range.

**Not here**: body weight. It is a time series (`BodyMetric`, feature 010), and a single
mutable column would make invariant 1's "history is not rewritten" impossible to state.

## Profile

Slow-changing descriptive data. One row per member, created with the member so no read path
has to handle its absence.

| Column | Type | Notes |
|---|---|---|
| `MemberId` | `BIGINT NOT NULL` | `FK_Profile_Member`, **restrict**, `UQ_Profile_MemberId` |
| `BirthYear` | `SMALLINT NULL` | `CK_Profile_BirthYear`, 1900 to the current year |
| `Sex` | `TINYINT NULL` | `0 Female`, `1 Male`, `2 PreferNotToSay` |
| `HeightCm` | `DECIMAL(5,2) NULL` | centimetres, canonical (invariant 4) |
| *(soft delete + audit as above)* | | |

One-to-one rather than columns on `Member`: `Member` is read on **every** authenticated
request (session resolution); `Profile` is read on one screen. Keeping them apart keeps the
hot row narrow.

`HeightCm` is `DECIMAL(5,2)`, never a float — `database-rules.md` and invariant 4. A member
who sets units to `lb / in` sees inches; the column does not change (FR-011).

## Session

What the BFF's cookie references and the API can revoke. Not a business entity: it is never
exposed, never listed, and has no member-facing identity.

| Column | Type | Notes |
|---|---|---|
| `Id` | `BIGINT IDENTITY` | |
| `MemberId` | `BIGINT NOT NULL` | `FK_Session_Member`, restrict; `IX_Session_MemberId` |
| `TokenHash` | `BINARY(32) NOT NULL` | `UQ_Session_TokenHash` — SHA-256 of the token |
| `CreatedAtUtc` | `DATETIME2(3) NOT NULL` | |
| `ExpiresAtUtc` | `DATETIME2(3) NOT NULL` | sliding, +14 days (auth contract §5) |
| `RevokedAtUtc` | `DATETIME2(3) NULL` | non-null means dead, whatever `ExpiresAtUtc` says |
| `LastSeenAtUtc` | `DATETIME2(3) NOT NULL` | written at most hourly |

**Three deliberate deviations from `database-rules.md`, approved in `plan.md` D3**:

1. **No `PublicId`.** The rulebook requires one on every *business entity*. A session has no
   external identity — the token is the only handle, and minting a second identifier for a
   secret-bearing row would create a way to name a session that no contract needs.
2. **No `CreatedBy` / `UpdatedBy`.** `MemberId` is the actor, and a session is written only
   by the authentication paths.
3. **No soft delete.** `RevokedAtUtc` is the tombstone, and dead rows are physically removed
   by the retention service. A session is not history: invariant 3 protects master data that
   a member's training log references, and nothing references a session. Keeping dead
   sessions forever would be a growing table of secret-adjacent rows for no benefit.

**Only the hash is stored.** A read of this table — a backup, a support query, a leak —
yields no usable session: SHA-256 is not reversible, and with 256 bits of entropy the token
is not searchable either. Lookup is by `TokenHash`; there is no other way in.

SHA-256 rather than the password hash of D2: this input is already high-entropy random, so
the slow salted hash a password needs would buy nothing and would cost that stretch on
**every authenticated request**. The two hashes protect different things and are chosen
separately — recorded because "use the strong one everywhere" is the plausible-sounding
mistake here.

## SignInAttempt

Backs the throttle (FR-016, auth contract §6). Append-only within its window; rows older
than the window are removed by the retention service.

| Column | Type | Notes |
|---|---|---|
| `Id` | `BIGINT IDENTITY` | |
| `NormalizedEmail` | `NVARCHAR(254) NOT NULL` | `IX_SignInAttempt_Email_At` |
| `SourceHash` | `BINARY(32) NOT NULL` | SHA-256 of the source address plus a server salt |
| `AttemptedAtUtc` | `DATETIME2(3) NOT NULL` | |

Rows are written for emails that **do not exist**, identically to ones that do. That is the
whole point: if the throttle only counted real accounts, 429-versus-401 would be the
existence oracle the decoy hash in auth §3 exists to close.

The source address is stored **hashed with a server-side salt**, not in the clear. An
address is personal data under invariant 10's "minimal", the throttle only ever needs
equality, and a table of members' home addresses beside their email is exactly the kind of
row that turns a small breach into a large one. The salt lives in configuration, never in
source.

## Persisted, per phase

| Phase | Migration | Content |
|---|---|---|
| 1 | `AddMemberAndProfile` | `Member`, `Profile` |
| 3 | `AddSessionAndSignInAttempt` | `Session`, `SignInAttempt` |

At most one migration per phase, named after what it adds (`database-rules.md`). Both are
purely additive — nothing is dropped, rewritten, or backfilled — so both down-paths are safe
(see `rollback.md`).

## Invariant trace

| Invariant | Where it binds here |
|---|---|
| 1 — completed training immutable | **Not applicable** — no `WorkoutSession` or `SetEntry` in this feature |
| 2 — one member, never leaves them | `Profile.MemberId` restrict FK; every contract path is `/me`-shaped, with no member parameter to get wrong |
| 3 — master data soft-deleted | No master data here. `Session` is not master data (see above) |
| 4 — canonical units and UTC | `HeightCm` decimal centimetres; every instant `DATETIME2(3)` UTC; `Units` is display-only and rewrites nothing |
| 5 — derived numbers not stored | Nothing derived is stored; `memberSince` is `CreatedAtUtc`, read directly |
| 6 — a set is physically possible | **Not applicable** — no `SetEntry` |
| 7 — rules live in the API | Every constraint above is in the API's schema; the BFF has no database connection |
| 8 — audit and identity | Audit fields on both business entities; `PublicId` is the only identifier on the wire |
| 9 — one authoritative state | **Not applicable** — no `WorkoutSession` or `Program` |
| 10 — minimal and deletable | No health data beyond height and birth year; soft delete then physical removal at 30 days; the source address is hashed, not stored |

Invariants 1, 6 and 9 are listed as *not applicable* rather than omitted, because the
Critical addendum's item 2 asks each review to say so for every item explicitly.
