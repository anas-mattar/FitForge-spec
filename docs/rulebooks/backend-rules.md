# Backend Rules — FitForge

> **Binding**: this rulebook is enforced through the backend tier's compliance checklist
> (Definition of Done item 5, `docs/sdlc/definition-of-done.md`): the checklist asserts,
> this rulebook explains. Domain invariants (`modules/training/training-invariants.md`) carry
> constitutional force and always outrank this file.

## Layering & Placement

- Code layout: fixed ADR-style in **specs/001-solution-scaffold/plan.md** (constitution IV,
  bootstrap clause). Until that feature merges there is no existing layout to follow, and no
  other feature may invent one. This line is rewritten with the actual layout in the same
  feature that establishes it.
- New code MUST follow the existing layout. A new top-level folder, layer, or project
  requires approval in the feature's `plan.md`.
  **Why**: architecture changes ride in on "just one helper folder" — this is where drift starts.

## API Surface

- Every external input MUST be validated at the API boundary; validation failures MUST
  return RFC 9457 problem details (`application/problem+json`).
  **Why**: the boundary is the one place a bad value is rejected once for all callers.
- Responses MUST use dedicated DTOs; domain entities MUST NOT be serialized directly.
  **Why**: entity leaks turn every schema refactor into a breaking API change.
- Code implementing a feature contract MUST cite the contract file (the feature's
  `contracts/` directory) in a comment at the top of the file.
  **Why**: the citation makes rung-checking possible during review.

## Domain Logic

- Domain invariants (`modules/training/training-invariants.md`) MUST be enforced in code AND asserted
  by the database schema where a constraint can express them — never in one place only.
- Money, dates, and numbers MUST be parsed and formatted culture-invariantly at every
  boundary. **Why**: a locale-dependent parse is data corruption that no test on the
  author's machine catches.
- The training calculations (set volume, estimated 1RM, weekly volume per muscle group,
  streak, adherence) MUST live in one named module in `fitforge-api` — `Training.Calculations`
  — covered by golden-fixture tests whose expected values are hand-worked from
  **docs/product/fitforge-logic.md** §4. Re-deriving them inline, anywhere, is prohibited.
  **Why**: two implementations of one formula always diverge, and here the divergence is a
  member's personal record changing depending on which screen they open.

## Data Access

- Read-only queries MUST NOT track entities (`AsNoTracking()` on every read path).
- Soft-delete filtering is owned by EF Core **global query filters** on every soft-deletable
  entity. A query that needs deleted rows MUST opt out explicitly with
  `IgnoreQueryFilters()`. **Why**: one opt-out that is visible in the diff beats a filter
  clause that has to be remembered in every query and is invisible when forgotten.
- Member scoping MUST NOT rely on the caller passing a member id: every training query
  derives the member from the authenticated principal.
  **Why**: invariant §2 is absolute, and a parameter can be tampered with.

## Testing

- xUnit. Every endpoint MUST have an API test covering its success path and its
  authorization failure. Every calculation in `Training.Calculations` MUST have a
  golden-fixture test with hand-worked expected values.
- Every invariant in **modules/training/training-invariants.md** that can be violated through
  the API MUST have a test that attempts the violation and asserts it is rejected.
  **Why**: an invariant with no attempted-violation test is a comment.
- The gate (`docs/sdlc/gate-command.md`) is run by the user; a phase is not done before
  the user confirms exit code 0 — or, under a declared `ci-held` (Lite/Micro/Standard
  only, constitution X), before the owner records approval on the CI evidence triplet.

## Security

- Secrets MUST NOT be committed. Configuration comes from `appsettings.json`; secrets come
  from user-secrets in development and environment variables in every deployed environment.
  `.env.example` and `appsettings.Example.json` carry names only, never values.
- Every training endpoint MUST require an authenticated member; anonymous access is opt-in
  per endpoint and reviewed as a security decision, never a default.
- Internal `Id` values MUST NOT appear in URLs, payloads, or logs — `PublicId` only
  (**modules/training/training-invariants.md** §8).
- Request bodies MUST have a size limit, and collection endpoints MUST be paged with a
  server-enforced maximum page size.
  **Why**: an unpaged list endpoint is a denial-of-service primitive a member can trigger
  by accident.
