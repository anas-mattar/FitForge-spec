# Integration Rules — FitForge

> **Binding**: contract-before-implementation is constitution VII; this rulebook only
> operationalizes it. It covers every call that crosses a repository or leaves the system
> (own API consumed by own frontend included).

## Contract First

- Every cross-repository or external call MUST have a written contract in the feature's
  `contracts/` directory BEFORE either side is implemented; both sides cite the contract
  file in a comment. **Why**: two teams (or two agents) coding against an imagined
  interface produce two interfaces.
- The contract outranks `tasks.md` and implementation notes (Source of Truth rung 4). If
  implementation and contract conflict: stop and report — never silently patch either.

## Change Discipline

- Additive changes (new optional fields, new endpoints) are the default evolution path.
  Breaking a published contract requires a spec update and `plan.md` approval, plus a
  coordinated cross-repository rollout per `docs/sdlc/repository-strategy.md`.
- Cross-repository order: backend implements and gates first; the frontend may proceed in
  parallel ONLY by mocking against the agreed contract, never against guesses.

## Wire Format

- camelCase JSON. List responses MUST be wrapped in a named object with the collection and
  its paging metadata, never returned as a bare array.
  **Why**: a bare array cannot grow a field without breaking every consumer.
- Instants are ISO 8601 UTC with an explicit `Z`; calendar dates are `yyyy-MM-dd`.
- Loads and body weights cross the wire in **kilograms** and lengths in **centimetres**, as
  decimal numbers with an explicit unit-bearing field name (`loadKg`, `heightCm`). Display
  units are a presentation concern and MUST NOT appear in a contract
  (**modules/training/training-invariants.md** §4).
  **Why**: a number named `load` is the bug; a number named `loadKg` cannot be silently
  misread by the other side.
- Identifiers on the wire are `PublicId` GUIDs. Internal `Id` values MUST NOT appear.
- Errors MUST use RFC 9457 problem details (`application/problem+json`) — consumers MUST
  distinguish "call failed" from "empty result".
  **Why**: a failure decoded as an empty list is silent data loss.

## Runtime Discipline

- Every outbound call MUST have an explicit timeout: 10 seconds for BFF-to-API reads,
  30 seconds for writes. There is no unbounded call in this product.
- Retry/backoff policy: no automatic retries. A write MUST NOT be retried automatically
  without an idempotency key agreed in the contract.
  **Why**: a silently retried "log this set" is a duplicated set, and invariant §1 makes a
  completed session immutable — so the duplicate is permanent.
- A failed integration MUST surface as the consumer's error/unavailable state — never as
  fabricated or cached-stale data presented as fresh.

## Configuration & Secrets

- Base URLs, keys, and credentials come from environment variables (user-secrets in local
  development), never hard-coded. Secrets MUST NOT reach client-side code: in `fitforge-web`
  only the BFF may read them, and no `NEXT_PUBLIC_`-prefixed variable may carry one.

## The BFF is not a tier that owns anything

- The `fitforge-web` BFF (`app/api/**`) MAY hold the member session, call `fitforge-api`,
  and aggregate or reshape responses for a screen. It MUST NOT open a database connection,
  own a domain rule, or become the only place a decision is made
  (**modules/training/training-invariants.md** §7).
- A BFF route that aggregates MUST NOT invent a field the API cannot produce. If a screen
  needs a value nothing serves, the contract changes — the BFF does not compute it.
  **Why**: a computed field in the BFF is a domain rule with no tests, no invariant
  coverage, and no reviewer looking for it.
