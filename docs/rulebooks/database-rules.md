# Database Rules — FitForge

> **Binding**: this rulebook is enforced through the compliance checklist of whichever
> tier owns the migration (Definition of Done item 5, `docs/sdlc/definition-of-done.md`).
> Schema changes are the least reversible thing an agent ships — this file is always read
> together with `docs/sdlc/rollback-process.md`, never alone.

## Setup

- Before the FIRST migration runs, the connection string's target database MUST be
  confirmed as dedicated to this project — not silently inherited from another project's
  local-dev setup and not a shared/production database. Deliberately sharing a database
  across services is permitted only as an explicit, plan-approved decision: the rule is
  confirmation, not a ban on reuse. **Why**: a connection string silently reused from
  another project points the first `migration apply` at an existing database, mixing
  schemas the moment it runs.

## Schema Standards

- Primary keys: the default primary key MUST be `Id BIGINT IDENTITY(1,1) PRIMARY KEY`
  (SQL Server). Every business entity ALSO carries `PublicId UNIQUEIDENTIFIER NOT NULL`
  with a unique index — the only identifier that may appear in a URL, payload, or log
  (**modules/training/training-invariants.md** §8). `BIGINT` rather than `INT` because
  `SetEntry` is the high-volume table and a member logs thousands of rows a year.
  Deviations are prohibited unless explicitly approved in the technical plan.
  Externally-exposed identifiers (public IDs, correlation IDs, integration references,
  idempotency keys) MAY use opaque values such as GUIDs, but these are not primary keys.
  **Why**: a uniform key strategy keeps indexes compact and joins predictable while still
  allowing opaque identifiers where external exposure genuinely requires them.
- Audit fields on every business entity: `CreatedAtUtc`, `CreatedBy`, `UpdatedAtUtc`,
  `UpdatedBy` — all instants stored UTC (**modules/training/training-invariants.md** §4). **Why**: systems of record require a verifiable trail of who
  changed what and when.
- Soft delete standard: `IsDeleted BIT NOT NULL DEFAULT 0`, `DeletedAtUtc`, `DeletedBy`,
  enforced by an EF Core global query filter; physical `DELETE` prohibited unless
  plan-approved. Business master data MUST use soft
  delete; physical deletion is prohibited unless explicitly approved in the technical plan.
  **Why**: master data referenced by history must never disappear from under it.
- Naming conventions: tables singular PascalCase (`WorkoutSession`), columns PascalCase,
  indexes `IX_<Table>_<Columns>`, foreign keys `FK_<Table>_<ReferencedTable>`, check
  constraints `CK_<Table>_<Rule>`, unique constraints `UQ_<Table>_<Columns>`.
- Loads and body weights are `DECIMAL(6,2)` kilograms; lengths `DECIMAL(5,2)` centimetres;
  RPE `DECIMAL(3,1)`. Floating-point types MUST NOT be used for any member-visible number.
  **Why**: invariant §4 requires canonical storage, and a float turns 100.0 into 99.99999.

## Constraints Mirror Invariants

- Every domain invariant (`modules/training/training-invariants.md`) that a constraint can express MUST
  be one — CHECK, FK, UNIQUE, NOT NULL — in addition to application-level enforcement.
  **Why**: application-only enforcement is one forgotten code path away from bad data.
- Foreign keys to soft-deleted master data MUST use restrict semantics, never cascade
  delete. **Why**: history must keep resolving after the referenced row is retired.

## Migrations

- Schema changes ship ONLY as EF Core migrations, generated and applied from
  `fitforge-api` — no hand-run SQL against shared environments, and no migration in any
  other repository (`fitforge-api` owns the database, exclusively). **Why**: unscripted changes cannot be replayed, diffed, or rolled back.
- At most one migration per phase, named after the feature.
- Every migration MUST be reversible, or ship a written rollback plan from
  `specs/_templates/rollback-template.md`. **Why**: rollback designed after the incident
  is guesswork.
- Destructive operations (dropping columns/tables, truncating, rewriting data) are
  prohibited unless explicitly approved in the feature's `plan.md`.
- A migration already applied beyond the author's machine MUST NOT be edited — write a
  new one. **Why**: edited history diverges environments silently.

## Data Safety

- Rolling back code or a migration MUST NOT cascade into physical deletion of domain
  data. If a rollback would touch domain data, stop and report
  (`docs/sdlc/rollback-process.md`).
- Seed data: deterministic only — the exercise/gear/muscle-group catalog seeds with fixed
  `PublicId` values so fixtures and screenshots stay stable across re-seeds. Seed data MUST
  NOT include member accounts or training history outside a test fixture.
  **Why**: a random seed makes every golden-fixture test and every visual reference a
  moving target.
