# Rollback — 002 Identity and Member Profile

**Filled 2026-09-10, before phase 1 begins** — `docs/sdlc/critical-delivery.md` item 1. A
rollback plan written after the change is a description, not a plan.

**Status of this document**: written against the plan, not against shipped code. Each phase's
commit sha is filled in as that phase lands, and the "Verification after rollback" checks are
executed only if a rollback actually happens.

## Rollback Method

```bash
git revert <phase-commit-sha>        # one revert per phase commit, newest first
```

Reverts are taken **newest first**, in both repositories, and the two repositories are
reverted in the reverse of the order they were built: web (phases 9, 8, 7) before API
(6, 5, 4, 3, 2, 1). Reverting the API first leaves the web application calling endpoints
that no longer exist.

| Phase | Repository | Commit | Reverting it alone leaves the tree correct? |
|---|---|---|---|
| 1 | fitforge-api | *(filled at phase 1)* | Yes, if 2–6 are already reverted |
| 2 | fitforge-api | | Yes, if 4–6 are already reverted |
| 3 | fitforge-api | | Yes, if 4–6 are already reverted |
| 4 | fitforge-api | | Yes, if 7 is already reverted |
| 5 | fitforge-api | | Yes, if 9 is already reverted |
| 6 | fitforge-api | | Yes — nothing depends on the purge |
| 7 | fitforge-web | | Yes, if 8 and 9 are already reverted |
| 8 | fitforge-web | | Yes, if 9 is already reverted |
| 9 | fitforge-web | | Yes |
| 10 | governance | | Yes — documents only |

**Plain revert is sufficient for code. It is not sufficient for the database** — see below.
That asymmetry is the whole reason this document is longer than 001's would have been.

## Changed Areas

**`fitforge-api`** — `src/FitForge.Domain/Members/`, `src/FitForge.Infrastructure/Persistence/`
(configurations and two migrations), `src/FitForge.Api/Features/Identity/`,
`src/FitForge.Api/Features/Me/`, `src/FitForge.Api/Hosting/Authentication/`,
`src/FitForge.Api/Hosting/Retention/`, and the matching test projects.

**`fitforge-web`** — `src/app/(auth)/`, `src/app/(app)/profile/`, `src/app/api/bff/`,
`src/lib/session.ts`, `src/lib/units.ts`, `src/components/ui/` (generated primitives),
`.env.example`.

**Governance** — this feature's directory only.

**Blast radius beyond the diff**: after phase 7, every authenticated route in the web
application depends on the cookie this feature introduces. There is no feature 003 yet, so
nothing outside 002 breaks — this is the last moment in FitForge's life when that is true,
and it is why a rollback here is cheap and a rollback of the same seam in six months is not.

## Database Rollback

- **Schema changes in this feature**: **additive**, in two migrations.
  - Phase 1 — `AddMemberAndProfile`: creates `Member`, `Profile`.
  - Phase 3 — `AddSessionAndSignInAttempt`: creates `Session`, `SignInAttempt`.
- **Migration down-path**: both `down` methods drop only the tables their `up` created.
  Neither migration alters, backfills, or rewrites a pre-existing column — there are no
  pre-existing tables to touch, this being FitForge's first schema. Both are therefore safe
  to run **in a development database**.
- **Protected domain data touched**: **none.** No `WorkoutSession`, `SetEntry`,
  `SessionAdjustment`, `BodyMetric`, `Exercise`, `Gear` or `Program` row exists yet or is
  referenced by this feature. Invariant 1 and invariant 3 cannot be violated by this
  rollback, and this is the one feature in the roadmap for which that will be true.

### The part that is not a `git revert`

**A `down` migration destroys member accounts.** In development that is the intent. In any
environment holding a real member, dropping `Member` is a physical deletion of personal
data, and `docs/sdlc/rollback-process.md` is explicit that physical deletion is not a
rollback mechanism.

So the rule for this feature is:

- **Development / local**: run `down`, or drop and recreate. No approval needed.
- **Any environment with a real member row**: **stop and report.** Revert the *code* to
  disable the surface; leave the tables in place. A member who registered did not consent to
  having their account deleted because we changed our minds about a release.

FitForge has no deployed environment as of 2026-09-10, so today the first branch applies. The
second is written now because it will be true before it is next read, and because
"we'll think about it when we deploy" is how the rule gets broken.

## Deployment Rollback

Nothing to undo in infrastructure — there is no deployment yet. What a revert must undo
alongside the code:

| Item | Where | Undo |
|---|---|---|
| `FITFORGE_SESSION_SECRET` **removed** by phase 7 | `fitforge-web/.env.example` | Reverting phase 7 restores the line. Harmless either way — nothing reads it (plan D4) |
| Source-address salt for the throttle | API configuration / environment | Added in phase 4. Remove it when phase 4 is reverted; it is a secret with no other consumer |
| Retention service enabled | API host | Runs on startup from phase 6. Reverting phase 6 removes it; no state is left behind, because the service holds none |
| No feature flag, no permission grant, no external registration | | Nothing |

**The retention service is the one item worth pausing on.** If it has already run and
permanently removed a soft-deleted member, that removal is not reversible by any revert
here — the rows are gone. This is the intended behaviour of invariant 10 and not a defect,
but a rollback plan that failed to say so would be hiding the single irreversible act this
feature can perform. Before reverting phase 6 in any environment that has run it, check
whether a purge has occurred and record what it removed.

## Verification After Rollback

- [ ] Gate passes on the reverted state — **user-confirmed exit code**, in both repositories.
      This feature is `**Gate Certification**: user-run` (Critical features may not be
      `ci-held`), so a green CI run does not certify the reverted state either.
- [ ] `GET /health/live` and `GET /health/ready` answer as they did before 002 — the
      pre-feature behaviour, restored (001's contract).
- [ ] The web application builds and the shell renders with no authentication surface: no
      sign-in route, no profile route, no `/api/bff/*` handler, and no session cookie set on
      any response.
- [ ] `Member`, `Profile`, `Session` and `SignInAttempt` are either absent (development) or
      present and untouched (any environment with a real member row) — never partially
      dropped.
- [ ] `git grep` finds no reference to the removed BFF routes from any remaining component;
      a dangling `fetch("/api/bff/me")` compiles happily and fails at runtime.
