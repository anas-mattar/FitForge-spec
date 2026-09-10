# Contract: Health — `fitforge-api` → `fitforge-web` (BFF)

**Status**: Agreed (constitution VII — this contract precedes implementation on both sides)
**Consumer**: `fitforge-web` BFF route handler
**Provider**: `fitforge-api`
**Feature**: 001-solution-scaffold

This is FitForge's first BFF↔API contract, so it also serves as the worked example every
later contract in `specs/NNN-*/contracts/` is written against. It is deliberately trivial in
content and complete in form.

## 1. Provider endpoints

### `GET /health/live` — liveness

Answers one question: is this process running and able to serve HTTP? It performs no
dependency work and MUST NOT touch the database.

- **200 OK** — the process is live.

```json
{ "status": "live" }
```

There is no failure response. A process that cannot answer is, by definition, not live; the
consumer observes that as a transport failure.

### `GET /health/ready` — readiness

Answers: can this process serve real requests right now? It checks each declared dependency.

- **200 OK** — every dependency is usable.
- **503 Service Unavailable** — at least one is not.

```json
{
  "status": "ready",
  "checks": [
    { "name": "database", "status": "ready", "durationMs": 12 }
  ]
}
```

`status` is `ready` or `degraded` at the top level, and `ready` or `failed` per check.
`checks[].name` is a stable machine identifier, lower-case, never a display string.

**Readiness MUST answer within 3 seconds**, including when a dependency is down.

This is not a performance target, it is what keeps `degraded` reachable. The consumer
applies a 10-second timeout (§3), so a readiness check that takes longer than that turns
every "the API is degraded" into "the API is unreachable" — the exact collapse §2
forbids. Each dependency check therefore carries its own timeout, and the check reports
`failed` on its own terms rather than letting the caller give up first.

The three seconds bound the **whole document**, so a single check's timeout must be
strictly shorter than it — a check bounded at exactly three seconds cannot fit inside a
three-second document and leaves nothing for the checks beside it. The database check is
bounded at two.

Found the hard way: `AddDbContextCheck` against an unreachable SQL Server takes ~15
seconds on a cold attempt and ~40ms once SqlClient has a cached failure, so the symptom
also flaps — the first probe after a quiet period says `unreachable`, the next says
`degraded`.

Readiness MUST NOT disclose connection strings, server names, credentials, exception
messages, or stack traces — a readiness probe is reachable by anything that can reach the
API. A failed check reports the name and the fact of failure, nothing more.

## 2. Consumer endpoint (BFF)

### `GET /api/health` — the only shape the browser sees

The browser calls this and nothing else. It never holds `FITFORGE_API_BASE_URL`, and the API
is never addressable from the browser.

- **200 OK** in all three cases below. Reachability is *data*, not an HTTP failure — the BFF
  succeeded at finding out.

```json
{ "api": "ready",       "checkedAt": "2026-09-10T01:23:45Z" }
{ "api": "degraded",    "checkedAt": "2026-09-10T01:23:45Z" }
{ "api": "unreachable", "checkedAt": "2026-09-10T01:23:45Z" }
```

Mapping, which is the whole of the BFF's logic here:

| Upstream observation | `api` |
|---|---|
| `/health/ready` → 200 | `ready` |
| `/health/ready` → 503 | `degraded` |
| timeout, connection refused, DNS failure, any non-200/503 | `unreachable` |

`degraded` and `unreachable` are distinct and MUST NOT be collapsed: one means the API
answered and told the truth about itself, the other means it did not answer. Conflating them
sends a developer to debug the wrong process.

## 3. Conventions this contract fixes for every later contract

- **Casing**: JSON is `camelCase` on both sides, always.
- **Timestamps**: UTC, ISO 8601, `Z`-suffixed. Never a local time, never an offset.
- **Durations**: integer milliseconds, suffixed `Ms`.
- **Errors**: every non-2xx from the API is an RFC 9457 problem document
  (`application/problem+json`), never framework default HTML — including 404 and 405.
- **Timeouts**: the BFF applies a 10-second timeout to this call and MUST NOT retry it. A
  health probe that retries reports a stale truth.
- **No client-side base URL**: the API's address exists only in the BFF's server-side
  environment.

## 4. Verification

- Provider: an automated test per endpoint asserting status code and body shape, including
  the 503 path with a dependency forced to fail.
- Consumer: an automated test per row of the mapping table above, including the timeout row.
- End to end (manual, once, recorded in the phase summary): stop the API, reload the page,
  observe `unreachable` and an intact shell.

## 5. Change policy

Changing this contract is a change to both repositories and MUST be agreed in the consuming
feature's `contracts/` before either side is edited. Adding a field is backward compatible;
renaming or removing one is not, and requires both repositories to ship in the same feature.
