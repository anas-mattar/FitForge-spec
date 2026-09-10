# Onboarding — working on FitForge

For the second developer joining the project, and for anyone returning after a break. It
assumes you have read nothing else yet.

## 1. Clone the nested layout

All three repositories, in this shape, in this order:

```bash
git clone <governance-repo-url> fitforge
cd fitforge
git clone <api-repo-url> fitforge-api
git clone <web-repo-url> fitforge-web
```

The code repositories live **inside** the governance repository's directory and are ignored
by it. They stay fully independent repositories — never submodules.

**Do not clone `fitforge-api` on its own somewhere else.** An AI agent reads `CLAUDE.md` from
the working directory and its ancestors: inside the nested layout it inherits the constitution
automatically, and outside it, it inherits nothing and improvises. Same for you.

The cost of nesting is that three histories share one directory tree. Before every commit,
know which repository you are in — `git rev-parse --show-toplevel` answers it.

## 2. Get both processes running

Nothing below asks you to talk to anybody. If a step here is not enough on its own, that is
a defect in this file — fix it in your first feature branch.

### What you need installed

| | Version | Why that one |
|---|---|---|
| .NET SDK | **10.0.202** | Pinned in `fitforge-api/global.json`. A different feature band fails the build rather than silently using another compiler. |
| Node.js | **22 LTS** (20.9+ works) | Next.js 16 with Turbopack. |
| SQL Server | any reachable instance, or none | Optional for the shell — see "Running without a database" below. |

Check with `dotnet --version` and `node -v` before going further. A wrong .NET SDK is the
one prerequisite that produces a confusing error instead of a clear one.

### The API

```bash
cd fitforge-api
dotnet restore
```

Then supply the connection string as an environment variable and run. Bash:

```bash
export Database__ConnectionString="Server=localhost;Database=FitForge;Trusted_Connection=True;TrustServerCertificate=True"
dotnet run --project src/FitForge.Api
```

PowerShell:

```powershell
$env:Database__ConnectionString = "Server=localhost;Database=FitForge;Trusted_Connection=True;TrustServerCertificate=True"
dotnet run --project src/FitForge.Api
```

The double underscore is not a typo — it is how .NET spells a configuration section
separator in an environment variable.

The connection string is **required to start**. Without it the process stops immediately with
a message naming the setting — that is deliberate, not a bug: a missing value should cost you
one message, not an afternoon of debugging a request that fails later for an
unrelated-looking reason. The value never goes in `appsettings.json`; only the name lives in
source.

> **Known gap.** That startup message also suggests `dotnet user-secrets`, which does not
> work yet: `FitForge.Api.csproj` has no `UserSecretsId`, so the command exits with
> *"Could not find the global property 'UserSecretsId'"*. Use the environment variable above
> until phase 7 fixes it (`specs/001-solution-scaffold/tasks.md`). Found by following this
> file, which is what it is for.

It listens on `http://localhost:5212`. Verify:

```bash
curl http://localhost:5212/health/live     # {"status":"live"}
curl -i http://localhost:5212/health/ready # 200 + {"status":"ready","checks":[...]}
```

### The web application

In a second terminal:

```bash
cd fitforge-web
npm ci
cp .env.example .env.local     # Windows: copy .env.example .env.local
```

Then set the one variable that matters in `.env.local`:

```ini
FITFORGE_API_BASE_URL=http://localhost:5212
```

Never prefix it `NEXT_PUBLIC_`. The browser is not allowed to know the API's address — the
BFF holds it, and `fitforge-web/src/lib/api-client.ts` is marked `server-only`, so importing
it from a client component fails the build rather than leaking the address into a bundle.

```bash
npm run dev
```

Open `http://localhost:3000`.

### What "working" looks like

1. The shell renders: sidebar on the left at 1024px and wider, a five-item bottom bar below
   that, sticky header at the top.
2. The theme toggle in the header switches light and dark, and the choice **survives a
   reload** — it is written to `localStorage` and applied by a blocking script before paint,
   so there is no flash of the wrong theme.
3. The header shows a small dot and **"API ready"**. That single indicator is the proof both
   halves are talking, and the network tab shows only `localhost:3000` — never the API's
   host.

If all three hold, you are running. Nothing else in this repository needs to work yet.

### Running without a database

You do not need SQL Server to work on the shell or on anything front-end. Give the API any
syntactically valid connection string pointing nowhere and start it anyway: it will run, and
the header will read **"API degraded"**.

That is a correct answer, not a failure. The three words mean three different things, and
they are the whole reason the health contract exists:

| The header says | It means | Go look at |
|---|---|---|
| API ready | the API answered and every dependency is usable | nothing |
| API degraded | the API answered and told you a dependency is down | the database |
| API unreachable | the API never answered | the API process, or `FITFORGE_API_BASE_URL` |

If you ever see `unreachable` while the API is plainly running, that is a real bug — it means
something took longer than the BFF's ten-second patience. It has happened once already
(`specs/001-solution-scaffold/tasks.md`, phase 6).

### The gate commands

Before you ask anyone to certify anything:

```bash
cd fitforge-api && dotnet build --warnaserror && dotnet test
cd fitforge-web && npm run lint && npm run typecheck && npm run build && npm test
```

Both must exit 0. You report the exit code; you never declare the gate passed.

### When it does not start

| Symptom | Cause |
|---|---|
| `Failed to bind to address http://localhost:5212` … *socket in a way forbidden by its access permissions* | Windows has reserved that port range (error 10013), nothing is using it. Run on another port: `dotnet run --project src/FitForge.Api -- --urls http://127.0.0.1:8412`, and point `FITFORGE_API_BASE_URL` at it. |
| API exits at once, naming `Database:ConnectionString` | Working as designed — set the user secret above. |
| `Invalid framework identifier ''` during restore | Almost never what it says. Check `Directory.Build.props` for an XML comment containing a double dash, which is invalid XML. |
| Header stuck on "Checking API…" | The BFF's own route is failing. Check the `npm run dev` terminal, not the API. |

## 3. Read, in this order

1. `.specify/memory/constitution.md` — the law. Twenty minutes, once.
2. `modules/training/training-invariants.md` — ten rules that outrank every feature you will
   ever write here.
3. `docs/sdlc/flow.md` — the ritual on one page.
4. `docs/sdlc/definition-of-done.md` — the six gates.
5. `docs/sdlc/team-workflow.md` — how two people share this without colliding.

Then, only when you are about to touch that area, the matching row of `CLAUDE.md`'s
Task-Scoped Reading table. Do not read the rulebooks front to back; read the one for the tier
in your hands.

The digests in `docs/digests/` are an orientation aid, not law. They never satisfy a
"read first" obligation.

## 4. Claiming a feature

```bash
pwsh -File scripts/claim-feature.ps1 -ShortName session-logging "Log sets during a workout"
```

The remote branch **is** the claim — push it immediately, before you start work. Then flip
that feature's roadmap row to `in progress` in a separate `docs/` commit on main, with the
spec path written in brackets until the feature merges. CI fails if a claimed feature's
roadmap row still says `idea`: a stale roadmap on main actively misleads the other developer,
which is why it is a machine check and not a courtesy.

One active feature each. If you are blocked, say so and pick up a review — do not start a
second feature to look busy.

## 5. The loop, per feature

Spec → plan → tasks → **one phase at a time**. Per phase:

1. Declare that phase's **Territory** in `tasks.md` — the files it may touch — and commit
   that declaration **before** you touch them. In this multi-repo project, Territory entries
   are repo-prefixed from the governance root: `fitforge-api/src/Domain/**`,
   `fitforge-web/app/(training)/**`.
2. Implement. Commit with `phase N` in the subject; that token is how the machine attributes
   the commit.
3. `pwsh -File scripts/ritual-checks.ps1` — every check, one command. `scope-check` grades the
   governance repository, `scope-repos` grades your commits in `fitforge-api` and
   `fitforge-web`. Neither may FAIL.
4. Ask the owner to run the gate, or — if the plan declares `ci-held` — report the CI evidence
   and request their approval. **You never declare the gate passed, and neither does your
   agent.**

Territory is never back-declared. A code commit that predates its own declaration fails the
check, deliberately: a territory written after the fact records what you did, not what you
agreed to do.

## 6. Review

Every feature gets an AI review by a **fresh context** — never the session that wrote the
code, which cannot see its own blind spots — with the Reviewer Provenance block filled in.
Then the other developer's human review. Anas reviews Ahmad's features; Ahmad reviews Anas's.
Nobody reviews their own, and nothing merges without the other's approval.

When a review disagrees with you, the disagreement is the point. Answer it in writing.

## 7. UI work

`docs/design/fitforge-prototype.html` is the source visual reference — open it in a browser
and press `A` to toggle the annotations. Copy the screens your feature implements into
`specs/NNN-name/screenshots/`, and grade against those with the Visual Compliance Loop
(`docs/sdlc/review-process.md`). Never invent a layout when a reference exists; if the
reference is wrong, change the reference in the same feature and say why.

## 8. Governance changes ride alone

A change to the constitution, `docs/sdlc/`, or a rulebook goes on its own `docs/` branch and
merges on its own. Never bundle a rule change with the feature that made you want it — that is
how a rule gets adopted without anyone reviewing it as a rule.
