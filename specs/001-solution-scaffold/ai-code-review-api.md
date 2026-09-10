# AI Code Review — 001 Solution Scaffold (fitforge-api)

**Reviewer**: fresh-context agent — claude-opus-5
**Date**: 2026-09-10
**Branches**: fitforge-api `001-solution-scaffold` (tip `cded3bd`) — phase commits `47ace55` (1),
`404a10b` (2), `cded3bd` (6). Governance repository read at `ae7c38d`; fitforge-web read only
where the contract's consumer half is load-bearing (`fitforge-web/src/lib/health.ts`).
**Scope reviewed**: the full diff of all three phase commits, plus the current state of every
file in `fitforge-api/src/**` and `fitforge-api/tests/**` (13 C# files, 5 project files,
`fitforge-api/Directory.Build.props`, `fitforge-api/FitForge.slnx`, `fitforge-api/global.json`,
both `appsettings` files, `fitforge-api/src/FitForge.Api/FitForge.Api.http`, both workflow
files). Law read: constitution, spec, plan (ADR-001 §4, package list §5),
tasks (Territory per phase), `specs/001-solution-scaffold/contracts/health.md`,
`modules/training/training-invariants.md`, `docs/rulebooks/backend-rules.md`,
`docs/rulebooks/database-rules.md`. Not reviewed: phases 3 and 4 (fitforge-web), phase 5
(governance), phase 7 (not implemented).
**Feature contract**: scaffold only — no domain entity, no migration, no seeded data; three
projects with references pointing one way; `FitForge.Domain` references zero packages; packages
limited to plan §5 as amended `26d9108`; every failure leaves as RFC 9457; readiness answers
within 3s and discloses nothing.

## Reviewer Provenance

- **Reviewer**: fresh-context agent — claude-opus-5 (no implementation context; given only the diffs, spec, plan, tasks, contract and rulebooks)
- **Implementer**: claude-opus-5 (the implementing session; its reasoning was NOT provided to this reviewer)
- **Inputs provided**: phase 1/2/6 diffs, full source and test tree, spec.md, plan.md, tasks.md, contracts/health.md, constitution, training-invariants, rulebooks
- **Attestation**: This reviewer did not produce the diff under review.

## Verdict

**REQUEST CHANGES** — the layering, the error contract and the phase-6 timeout mechanism are
all real and all work: the three-project shape and its one-way references are as ADR-001
declares, `--warnaserror` builds clean with zero warnings, 22 tests pass, the scope check PASSes
every phase commit, and I measured the production registration actually carrying a 2-second
bound. Three things must change before merge. First, `GET /health/ready` emits a
self-contradictory document — I observed HTTP **200** with `{"status":"ready"}` while the same
body reported the database check as `"failed"` — and the one test that constructs that exact
state asserts only the field that happens to be right. Second, `ConfigurationTests` fails on any
machine where `Database__ConnectionString` is set, which is precisely the machine
`docs/onboarding.md` §2 tells a developer to create; the gate is therefore not reproducible for
the developer SC-001 is measured on, and CI cannot reveal this because a runner has no such
variable. Third, the test the ADR, the phase 1 commit message and the `FitForge.Domain` csproj
all cite as the mechanical guard on training invariants 1 and 5 does not test what its name
says: I added `Newtonsoft.Json` **and** `Microsoft.EntityFrameworkCore` as package references to
`FitForge.Domain` and all four domain tests stayed green. Residual risk sits in the readiness
endpoint's status mapping (the shape every later health check inherits) and in the fact that
the phase-6 bound is silently discardable by any later options configuration.

## What was verified (evidence)

| Area | Evidence |
|---|---|
| Spec match (FRs implemented as specified) | FR-001 both endpoints exist and are distinct (`fitforge-api/src/FitForge.Api/Features/Health/HealthEndpoints.cs`, lines 25 and 31); liveness proven not to consult dependencies by a test hosting a failing check (`fitforge-api/tests/FitForge.Api.Tests/HealthLiveTests.cs` lines 45-59). FR-002 404 and 405 observed as `application/problem+json` (`ProblemDetailsTests`, 2 tests green). FR-003 verified by reading all five csproj files: `FitForge.Domain` has no `ProjectReference`; `FitForge.Infrastructure` references Domain only; `FitForge.Api` references both; nothing references `FitForge.Api` except the test project. FR-004 verified by grep — `SqlConnection\|SqlCommand\|FromSql\|ExecuteSql\|DbContext\|UseSqlServer` over `src/FitForge.Api` and `src/FitForge.Domain` returns nothing. FR-011 **FAILS in the documented developer environment** — see F2. FR-005…FR-010, FR-013 are fitforge-web and out of this review's scope. |
| Visual-reference match | N/A — no UI ships in `fitforge-api`. Phase 3's Visual Compliance Loop table is recorded in `specs/001-solution-scaffold/tasks.md` and belongs to the fitforge-web review. |
| Feature contract held (no unapproved table/migration/permission/package) | No migration and no `DbSet` — `fitforge-api/src/FitForge.Infrastructure/Persistence/FitForgeDbContext.cs` is empty of both. Package audit against plan §5 as amended: `Microsoft.EntityFrameworkCore.SqlServer`, `Microsoft.EntityFrameworkCore.Design`, `Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore` — all approved and all confined to `FitForge.Infrastructure`; `Microsoft.AspNetCore.Mvc.Testing` and `xunit` approved. Four references sit outside the list: `Microsoft.AspNetCore.OpenApi` and the `coverlet.collector` / `Microsoft.NET.Test.Sdk` / `xunit.runner.visualstudio` trio — see F9. `dotnet list package --include-transitive` confirms the third-party SQL-client duplication the §5 amendment was written to avoid does not exist: one `Microsoft.Data.SqlClient` 6.1.6, pulled by EF Core. |
| Constitution / domain invariants | V: no domain rule ships, so none can be violated; the structural guard the ADR relies on is weaker than claimed (F3). VI: no secret in source — `appsettings.json` carries the key with an empty value and a `_comment` naming the two supply routes; no credential in either workflow file. VII: the contract exists and predates both sides, but omits three elements the principle lists as MUST (F7). VIII: coverage exists for the two things the plan named as logic — the readiness failure path and the timeout path. X: one phase per commit, `phase N` in every subject, `ci-held` declared in `plan.md` line 6 before phase 1, and every commit message ends with "Agent-run feedback (NOT certification)" rather than a success claim. II: an unreported internal conflict in the contract was resolved silently (F6). |
| Security (authn/authz, secrets, sensitive logging) | Disclosure verified by execution, not by reading: with a stubbed failure carrying `Login failed for user 'sa' on server db-prod-01.internal` in both Description and Exception, the observed 503 body was exactly `{"status":"degraded","checks":[{"name":"database","status":"failed","durationMs":0}]}`. On the **timeout** path — where the framework writes `A timeout occurred while running check.` into Description and an `OperationCanceledException` into Exception — the observed body was byte-identical in shape. `DomainExceptionHandler` line 48 gates Detail on `exception is DomainException || environment.IsDevelopment()`, and the non-development 500 path is asserted against the raw JSON. No authentication anywhere; spec Out of Scope covers it, but the contract never states that `/health/ready` is unauthenticated (F7). |
| Scope guard | `pwsh -File scripts/scope-check-repos.ps1 -All` from the governance root: `PASS phase 1 commit 47ace55 (18 file(s))`, `PASS phase 2 commit 404a10b (13 file(s))`, `PASS phase 6 commit cded3bd (3 file(s))`. Default invocation exits 0. `git show --stat` read on all three: phase 1 and 2 are coherent single slices; phase 6 is three files and touches nothing but the bound and its tests. Governance ordering checked by committer date — the §5 package amendment `26d9108` (10:20:34) predates phase 2 (10:25:55), and the two-second amendment `7d3f297` (11:15:03) predates phase 6 (11:15:32). Two declared Territory entries are dead paths (F10). |
| Rollback safety | No schema, no migration, no data. `git revert cded3bd` restores an unbounded check and a compiling tree — the phase is independently revertible as `plan.md` §Constitution Check claims. Reverting phase 2 would leave phase 1's `Program.cs` calling `AddFitForgePersistence`, so phases 2 and 6 revert as a pair, which matches the plan's own sizing note. |

## Findings

### F1 — `/health/ready` answers 200 "ready" while reporting a failed check — BLOCKING

`fitforge-api/src/FitForge.Api/Features/Health/HealthEndpoints.cs` maps the framework's three
`HealthStatus` values onto the contract's two words twice, with two different rules. Line 36
sets the top-level word from `report.Status == HealthStatus.Unhealthy ? "degraded" : "ready"`,
and line 50 chooses the status code from the same test — but line 42 sets each check's word
from `entry.Value.Status == HealthStatus.Healthy ? "ready" : "failed"`. `Healthy` is the
positive test in one place and `Unhealthy` the negative test in the other, so `Degraded` falls
on opposite sides of the two.

`HealthReport.Status` is the worst of its entries, so one `Degraded` entry makes the report
`Degraded`, which is not `Unhealthy`. Observed, by hosting the application with the existing
factory set to `HealthStatus.Degraded` and printing the real response:

```text
status=200 contentType=application/json; charset=utf-8
body={"status":"ready","checks":[{"name":"database","status":"failed","durationMs":0}]}
```

That document contradicts itself, and `specs/001-solution-scaffold/contracts/health.md` §1 is
explicit that 200 means "every dependency is usable". Worse, the BFF branches on the status
code alone (`fitforge-web/src/lib/health.ts`, `classifyStatus`), so a 200 becomes `ready` in the
browser while a dependency is unusable — the same collapse phase 6 exists to prevent, in the
other direction.

The state is not reachable through today's single check: `AddDbContextCheck` returns `Healthy`
or its `failureStatus`, which is `Unhealthy`, and the framework uses `FailureStatus` on timeout
too. It becomes reachable the moment feature 002 adds a second check, and this endpoint is the
shape every later check inherits.

What makes this a finding rather than a latent nit is the test.
`fitforge-api/tests/FitForge.Api.Tests/HealthReadyTests.cs` lines 52-66,
`A_degraded_dependency_is_not_reported_as_ready`, constructs exactly this state, and asserts
only `checks[0].status == "failed"`. It asserts nothing about the status code and nothing about
the top-level `status` — the two things its own name claims. It is green today and would stay
green through the whole defect. Its comment says "reporting a Degraded dependency as 'ready'
would tell the BFF everything is fine while a query is timing out"; that is precisely what the
code does.

*Action: implementer — make both decisions turn on the same test (`report.Status != HealthStatus.Healthy` for the word and the status code, matching line 42's per-check rule), and extend the existing test to assert 503 and top-level `degraded` so the name becomes true.*

### F2 — the gate fails on the machine `docs/onboarding.md` tells the developer to build — BLOCKING

`fitforge-api/tests/FitForge.Api.Tests/ConfigurationTests.cs` line 20 constructs a bare
`WebApplicationFactory<Program>` with no configuration override, on the stated reasoning that
"appsettings.json ships the key with an empty value, which is exactly the state a developer who
has not set the environment variable is in". The host also reads environment variables, at
higher precedence than `appsettings.json`. Reproduced:

```text
$ Database__ConnectionString="Server=x;Database=y;" dotnet test --filter ConfigurationTests
Failed FitForge.Api.Tests.ConfigurationTests.Startup_without_a_connection_string_fails_and_names_the_setting
   Assert.ThrowsAny() Failure: No exception was thrown
Failed! - Failed: 1, Passed: 1
```

`docs/onboarding.md` lines 53 and 60 instruct the developer to
`export Database__ConnectionString=...` / `$env:Database__ConnectionString = ...`. A developer
who follows onboarding and then runs the gate in that shell gets a red suite. That is FR-011
("each repository's gate command MUST exit 0 on the delivered baseline") failing against SC-001
("clone to both processes running by following `docs/onboarding.md` alone, with no question
asked of another person") — the first question they will ask is why the tests are broken.

CI cannot surface this: a GitHub runner has no such variable, so `project-gate.yml` is green.
Under `ci-held` certification the owner would be approving an evidence triplet that is true of
the runner and false of both developers' machines.

*Action: implementer — make the test independent of the ambient environment (clear the configuration sources, or override `Database:ConnectionString` to empty in that factory) so it asserts the empty-value state it says it asserts.*

### F3 — the domain-purity test does not test what its name, the ADR and the commit message all claim — BLOCKING

Three documents make the same load-bearing claim. ADR-001 §4.3: "Enforced by the absence of the
package reference". `fitforge-api/src/FitForge.Domain/FitForge.Domain.csproj` lines 3-14: "A
project with no package reference is the only guard against that which does not depend on
somebody remembering". Phase 1's commit message: "If someone adds a package to
`FitForge.Domain`, the test fails before review does."

`fitforge-api/tests/FitForge.Domain.Tests/DomainExceptionTests.cs` lines 45-60,
`The_domain_assembly_references_no_third_party_package`, reads
`Assembly.GetReferencedAssemblies()`. That is the emitted assembly-reference table, which the
compiler populates only from types the code actually uses — not the project's package
references. Tested on a scratch copy of the tree:

```text
# FitForge.Domain.csproj + <PackageReference Include="Newtonsoft.Json" Version="13.0.4" />
#                        + <PackageReference Include="Microsoft.EntityFrameworkCore" Version="10.0.12" />
Passed!  - Failed: 0, Passed: 4, Skipped: 0, Total: 4
```

Both packages restore into the project and all four domain tests stay green. Adding one file
that *uses* a type (`typeof(Microsoft.EntityFrameworkCore.DbContext).Name`) does turn it red —
so the test catches usage, and only usage. The gap is real and is exactly the shape the guard is
meant to close: someone adds EF Core to `FitForge.Domain` in one commit and writes the
calculation that uses it in the next, and only the second commit is caught.

The test is worth keeping — it is the last line. But the claim that a package reference "fails
before review does" is false, and it is repeated in three places where a later reader will rely
on it.

*Action: implementer — either assert against the project file itself (no `PackageReference` element in `FitForge.Domain.csproj`) so the claim becomes true, or rename the test to what it actually asserts and correct the three documents that overstate it. The owner should not have to discover which one is true.*

### F4 — the runtime guard checks the loose relationship and never the binding one — CONFIRM

T048 requires the bound to be asserted "in `DependencyInjection`, not only in configuration".
`fitforge-api/src/FitForge.Infrastructure/DependencyInjection.cs` line 64 does that with
`if (DatabaseHealthCheckTimeout >= ReadinessConsumerTimeout)` — 2 seconds against 10. But the
binding constraint, and the entire reason the value is 2 and not 3, is the *document* budget.
`ReadinessDocumentBudget` is declared at line 48 with a doc comment stating "Every check's own
timeout must be strictly shorter than this" — and then never used. Grep over the whole
repository:

```text
src/FitForge.Infrastructure/DependencyInjection.cs:48  (declaration)
tests/FitForge.Api.Tests/HealthReadyTimeoutTests.cs:55 (the only consumer)
tests/FitForge.Api.Tests/HealthReadyTimeoutTests.cs:57 (its message)
```

So a future edit to `TimeSpan.FromSeconds(5)` passes the runtime guard (5 < 10), starts, serves,
and breaks the 3-second contract bound — reintroducing a weaker form of the defect phase 6 was
created to fix. Only `HealthReadyTimeoutTests` line 54 stops it.

Two further problems with the guard as written. It is unreachable: both operands are
`static readonly TimeSpan` initialised from literals fourteen lines apart in the same file, so
the condition cannot be true at runtime without an edit to that file, and no test covers the
`throw`. And T048's stated rationale — "so it cannot be widened past the contract's consumer
timeout by an appsettings edit" — describes a threat that does not exist: the timeout is not
bound to configuration at all, and there is no code path by which `appsettings.json` can reach
it.

This is a CONFIRM rather than a BLOCKING because there is a legitimate case for deleting the
`throw` entirely — the test asserts both relationships, at build time, in every environment,
which is strictly better than a startup exception — and that is the owner's call, not mine.

*Action: owner — decide between (a) extending the guard to also compare against `ReadinessDocumentBudget`, or (b) deleting the guard and letting `HealthReadyTimeoutTests` own the relationship. Either way T048 should be re-worded: its "appsettings edit" premise is not true of the code as built.*

### F5 — the comment justifying the loop describes a state that cannot occur, and the loop fails silently — MINOR

`fitforge-api/src/FitForge.Infrastructure/DependencyInjection.cs` lines 103-105: "Written as a
loop over matches rather than `Single()` so that a test host which has already replaced the
registrations is not made to crash here."

`IConfigureOptions<T>` delegates run in the order they were added to the service collection.
This one is added inside `AddFitForgePersistence`, so it always runs *before* anything the host
or a test adds afterwards — a test host cannot have "already replaced" the registrations at the
moment it executes. `Single()` and the loop are indistinguishable in the composition the code
actually has. Measured on the real composition:

```text
AddFitForgePersistence alone                          -> database=timeout:00:00:02
AddFitForgePersistence + a later Configure that
  clears and re-adds the "database" registration      -> database=timeout:-00:00:00.0010000
```

The second value is `Timeout.InfiniteTimeSpan`. The direction of fragility is the opposite of
the one the comment guards: a delegate registered *after* this one silently discards the bound,
with no exception, no log line and no failing test. That is not hypothetical —
`fitforge-api/tests/FitForge.Api.Tests/FitForgeApiFactory.cs` lines 58-69 does exactly this, so
**every `WebApplicationFactory`-hosted test in the suite runs with the phase-6 bound removed**.
The only thing holding the production value is `ResolveDatabaseRegistration`
(`fitforge-api/tests/FitForge.Api.Tests/HealthReadyTimeoutTests.cs` lines 119-131), which builds
a bare `ServiceCollection` — a good choice, and the sole one.

The mechanism itself is correct and I confirmed it produces 2 seconds. The defect is a comment
that will teach the next maintainer something false about the one thing this phase installed.

*Action: implementer — correct the comment to state the real ordering property, and consider making the no-match case loud (the loop currently cannot distinguish "applied" from "found nothing").*

### F6 — the contract contradicts itself on the 503 media type, and the conflict was resolved silently — CONFIRM

`specs/001-solution-scaffold/contracts/health.md` §3 states: "every non-2xx from the API is an
RFC 9457 problem document (`application/problem+json`), never framework default HTML —
including 404 and 405." §1 defines the 503 readiness response as the readiness document. Both
cannot hold. Observed on the 503 path:

```text
status=503 contentType=application/json; charset=utf-8
```

The implementation chose §1, which I think is the right choice — a machine-readable readiness
document is more useful to the BFF than a problem document, and the BFF only reads the status
code. But constitution II requires that a conflict between rungs stops work and is reported, not
silently resolved, and nothing in the phase 2 commit message, `plan.md` or `tasks.md` records
that this one was noticed. This contract is also billed as "the worked example every later
contract in `specs/NNN-*/contracts/` is written against", so the ambiguity propagates.

*Action: owner — amend §3 to carve out the health documents §1 defines ("every non-2xx **error** response"), so the next contract inherits a rule that is true.*

### F7 — the contract omits three elements constitution VII lists as MUST — CONFIRM

Constitution VII: "Each contract MUST define purpose, authentication, endpoints, request schema,
response schema, error schema, timeout policy, retry policy, idempotency strategy, and audit
requirements." `specs/001-solution-scaffold/contracts/health.md` covers purpose, endpoints,
response schema, error schema, timeout policy and retry policy. It does not state
**authentication** (both endpoints are anonymous — a deliberate and probably correct decision
for a probe, but an unstated one), **idempotency strategy** (trivially satisfied by two GETs,
but the contract says so nowhere), or **audit requirements** (none, presumably). The document
asserts of itself that it is "deliberately trivial in content and complete in form"; it is not
complete in form, and it is the template every later contract copies.

*Action: owner — add the three headings with their trivial answers, so the worked example teaches the full shape.*

### F8 — T017's README does not exist — MINOR

T017: "Add the connection-string **name** to `appsettings.json` with an empty value and document
the environment variable in the repository README." The first half is done
(`fitforge-api/src/FitForge.Api/appsettings.json` lines 9-12). The second half is not: there is
no `README.md` anywhere in `fitforge-api` — the repository root holds only
`.github`, `.gitignore`, `Directory.Build.props`, `FitForge.slnx`, `global.json`, `src`, `tests`.
The information exists in three other places (the `_comment` key, `DatabaseOptions.MissingConnectionStringMessage`,
and `docs/onboarding.md`), so nobody is stranded — but a developer who opens the code repository
first finds nothing at all, and the task is recorded as part of a certified phase.

*Action: implementer — add the README in the phase 7 commit (its Territory already covers `fitforge-api/src/**` only, so this needs a Territory line first), or strike T017 with a note that `docs/onboarding.md` is the single home.*

### F9 — four package references sit outside plan §5 — MINOR

Plan §5 says "Nothing outside this list may be added without amending this plan (constitution
IV)". Four references are not on it:

- `Microsoft.AspNetCore.OpenApi` 10.0.12 in `FitForge.Api` — present in the pre-feature scaffold `8bfd51b`, so not an addition, but phase 1 rewrote that csproj and kept it.
- `coverlet.collector`, `Microsoft.NET.Test.Sdk`, `xunit.runner.visualstudio` — also scaffold defaults in `FitForge.Api.Tests`, but phase 1 created `fitforge-api/tests/FitForge.Domain.Tests/FitForge.Domain.Tests.csproj` **new** with all three. In that project they are additions.

Nothing here is a substantive risk — they are the xUnit template's runner and the ASP.NET
scaffold's OpenAPI generator. The problem is that feature 002 will create a test project and hit
the same question, and §5 as written says the answer is no.

*Action: owner — amend §5 to name the test-harness set (`Microsoft.NET.Test.Sdk`, `xunit.runner.visualstudio`, `coverlet.collector`) and `Microsoft.AspNetCore.OpenApi` as carried over from the scaffold.*

### F10 — two declared Territory entries name paths that do not exist — MINOR

Phase 2's Territory in `specs/001-solution-scaffold/tasks.md` declares
`fitforge-api/appsettings.json` and `fitforge-api/appsettings.Development.json`. Both files live
at `fitforge-api/src/FitForge.Api/`. The two entries match nothing and never could; the commit
passed only because `fitforge-api/src/**` already covers them. Dead territory is not a scope
violation, but it is a declaration written without checking, and the scope check cannot tell the
difference between an entry that is redundant and one that is a typo.

*Action: implementer — drop the two entries, or correct them to the real paths, in a governance commit.*

### F11 — the timeout disclosure test cannot fail — MINOR

`fitforge-api/tests/FitForge.Api.Tests/HealthReadyTimeoutTests.cs` lines 94-112,
`A_check_that_overruns_discloses_nothing_about_the_server`, asserts the body contains neither
"timeout occurred" nor "Exception". The endpoint projects each entry into a fixed three-field
record (`name`, `status`, `durationMs`) and never reads `Description` or `Exception` — so on the
timeout path the body is character-for-character the same shape as on the ordinary failure path,
which `HealthReadyTests` already covers. There is no code path by which either string could
appear. I confirmed the two bodies are identical in shape by observing both.

It is not worthless: it would catch someone adding a `description` field to `ReadinessCheck`
later. But it is presented as scrutiny of a distinct risk, and the risk is not distinct. The
`DoesNotContain("Exception", ..., OrdinalIgnoreCase)` assertion is also brittle in a way its
author probably did not intend — it will fire on any future check whose *name* contains that
substring.

*Action: none — note for reviewer awareness. If it is kept, its comment should say it guards the projection, not the timeout path specifically.*

### F12 — `AddFitForgePersistence` ignores the `IConfiguration` it is handed — MINOR

`fitforge-api/src/FitForge.Infrastructure/DependencyInjection.cs` line 59 takes an
`IConfiguration configuration` parameter. Grep for `configuration` in that file returns exactly
one hit: the parameter declaration. The options block at lines 80-88 uses
`.Configure<IConfiguration>((options, config) => ...)`, which resolves `IConfiguration` from the
service provider instead. Two consequences: a caller who passes a section or a test-specific
configuration is silently ignored, and the method carries an undeclared requirement that
`IConfiguration` be registered in DI — which is why
`fitforge-api/tests/FitForge.Api.Tests/HealthReadyTimeoutTests.cs` line 122 has to add it by hand
*and* pass one at line 124. The signature is a lie about where the configuration comes from.

On the brief's question of whether avoiding `.Bind()` is a legitimate constraint or a smell: the
constraint is real. `dotnet list package --include-transitive` on `FitForge.Infrastructure`
shows `Microsoft.Extensions.Configuration.Abstractions` and `Microsoft.Extensions.Options` in
the closure but neither `Microsoft.Extensions.Options.ConfigurationExtensions` (which supplies
`OptionsBuilder.Bind`) nor `Microsoft.Extensions.Configuration.Binder`, so `.Bind()` would
genuinely require a package plan §5 does not approve. Respecting that was right. The workaround
chosen was not the cheapest one available: reading the passed `configuration` parameter directly
needs no package either, uses the argument the method already takes, and removes the hidden DI
dependency. The comment at lines 75-79 defends the decision not to add a package, which nobody
would dispute, and never addresses the decision it actually made.

*Action: implementer — read the `configuration` parameter, or drop it from the signature. Passing an argument that is discarded is worse than not taking one.*

### F13 — XML documentation is never compiled, so every `see cref` in these files is unchecked — MINOR

`fitforge-api/Directory.Build.props` line 13 sets `GenerateDocumentationFile` to false. The
codebase is unusually heavily XML-documented — several files carry more doc comment than code —
and includes many `<see cref="..."/>` references (for instance `FitForgeApiFactory` refers to
`ConfigurationTests`, `DatabaseTimeout` and `AddFitForgePersistence`). With documentation
generation off, the compiler emits no CS1574 (unresolved cref) and no CS1570 (malformed XML), so
none of those references is verified by the `--warnaserror` build the gate leans on. A renamed
type leaves a dangling cross-reference that nothing reports.

*Action: none required for this phase — note for the owner. Turning it on would make the doc comments load-bearing in the same way the code is.*

## Constitution re-check (post-implementation)

**FAIL** — one principle is not satisfied as built.

- **I Specification First** — PASS. `spec.md`, `plan.md`, `tasks.md` and the contract all predate the first code commit (`5a34b5d` 09:48 vs `47ace55` 10:09), and phases 6 and 7 were each queued in governance before any code was written for them.
- **II Source of Truth** — **FAIL**. Not through a rung-ordering violation but through the conflict rule: the contract contradicts itself between §1 and §3 (F6), and the implementation chose one side without stopping and reporting.
- **III Repository Separation** — PASS. All three phase commits touch `fitforge-api` alone.
- **IV Architecture Consistency (bootstrap clause)** — PASS with a caveat. ADR-001's three-project shape, one-way references and `Features/<Area>/` layout are all present and verified. Four package references fall outside §5 (F9).
- **V Domain Invariants** — engaged only structurally, and more weakly than documented (F3). No domain rule ships, so none can be violated today.
- **VI Security** — PASS. No secret in source; disclosure verified by execution on both the failure and the timeout paths; the 500 path is asserted not to leak outside Development.
- **VII External Integration Governance** — **partial**. The contract exists and preceded both implementations, which is the substance of the principle; three of its ten mandated elements are absent (F7).
- **VIII Testing Requirements** — PASS. The two things `plan.md` §Constitution Check named as logic both have coverage. No business-critical calculation exists yet, so no golden fixtures are due.
- **IX Human Review** — pending. `specs/001-solution-scaffold/human-pr-review.md` is prepared and unticked, as it should be.
- **X Controlled Delivery** — PASS. One phase per commit, `phase N` in every subject, `ci-held` declared before phase 1, and no commit claims the gate — each says "Agent-run feedback (NOT certification)". Phase 6 correctly became its own phase rather than an amendment to certified work.

## Test coverage observed

`dotnet build --warnaserror` — Build succeeded, **0 Warning(s), 0 Error(s)**.
`dotnet test` — **22 passed, 0 failed** (Domain.Tests 4, Api.Tests 18).

**FitForge.Domain.Tests (4)** — exception hierarchy; `NotFoundException.For` message content; a
reflection sweep asserting no `DomainException` subtype exposes a `Status`/`HttpCode` property
(a genuinely good structural assertion — it would catch a real drift); and the domain-purity
test, which does not assert what it claims (F3).

**FitForge.Api.Tests (18)** —

- *Liveness (2)*: exact single-field body, and a 200 while the host's database check is failing. The second is the right test for "MUST NOT touch the database" and is not trivially true.
- *Readiness (4)*: healthy shape, the 503 `degraded` path, the `Degraded`-check case (asserts the one field that is right and skips the two that are wrong — F1), and a disclosure test asserting the raw document against a planted server name, login and exception type name. The disclosure test is the strongest in the suite: it asserts on the whole body rather than a property, so a leak in a field nobody anticipated still fails it.
- *Timeout (3)*: the registration relationship (both `< consumer` and `< document budget`, and — importantly — an explicit `NotEqual(Timeout.InfiniteTimeSpan)` first, because the unset value is negative and would pass a naive comparison; that guard is correct and easy to have missed); an end-to-end overrun producing 503 `degraded` in under 10s; and the timeout-path disclosure test that cannot fail (F11).
- *Problem details (7)*: 404 and 405 media types; a theory over the two domain-exception mappings; the domain message being returned deliberately; the non-development 500 asserted against the raw JSON for message, type name and stack frame; and the development case asserted to disclose. Exercising the handler directly rather than through a throwing route is the right call — the application ships no test-only endpoint.
- *Configuration (2)*: missing connection string naming the setting in all three forms (fails under the documented developer environment — F2), and startup succeeding without a reachable server.

What is not covered: no test measures the *whole readiness document* against the contract's
3-second budget in the production composition — T049 and T050 assert the registration's number
and a stub's overrun, and the 3.03-second measurement that set the value to 2 was manual and is
recorded only in a commit message. Nothing asserts the 200-path media type or that the readiness
body carries no extra field (liveness does assert this, via `Assert.Single`, which is why the
liveness test is stronger than the readiness one).

## Residual risk

The risk concentrates in two places, both of which the F-list names.

**The readiness status mapping (F1).** This is a scaffold whose whole purpose is that feature
002 inherits it. The endpoint's two-rule mapping is wrong in a way that is invisible today
because the only check cannot produce the state, and the test that could have caught it looks
at the one field the bug does not touch. Every health check added after this one arrives into a
projection that can emit `{"status":"ready"}` beside `"status":"failed"`. Fix before merge.

**Silent loss of the phase-6 bound (F5).** The bound is applied by mutating a shared
registration object from an options delegate. It works — measured at 2 seconds — but any later
`Configure<HealthCheckServiceOptions>` discards it without a sound, and one already exists in
the test factory. There is a single test standing between the project and a silent regression to
the ~15-second cold path, and it is a test that builds its own container rather than the host.
That is a thin thread for the defect the phase was created to fix.

Two smaller risks worth carrying forward. The internal response records are serialized
reflectively; a later move to source-generated JSON or AOT publishing would break the contract
bodies quietly, because no test asserts the wire shape from outside the process. And the
consumer's 10-second timeout is duplicated as a literal in two repositories
(`ReadinessConsumerTimeout` here, `HEALTH_TIMEOUT_MS` in `fitforge-web/src/lib/health.ts`) with
no mechanical link — they agree today, and nothing will notice the day they stop.

Merge after F1, F2 and F3 are fixed and the four CONFIRMs (F4, F6, F7, F9) have an owner
decision. F5's comment correction should ride with F1 since both touch the same phase's
reasoning. Phase 7 remains queued and out of this review.
