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

## 2. Read, in this order

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

## 3. Claiming a feature

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

## 4. The loop, per feature

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

## 5. Review

Every feature gets an AI review by a **fresh context** — never the session that wrote the
code, which cannot see its own blind spots — with the Reviewer Provenance block filled in.
Then the other developer's human review. Anas reviews Ahmad's features; Ahmad reviews Anas's.
Nobody reviews their own, and nothing merges without the other's approval.

When a review disagrees with you, the disagreement is the point. Answer it in writing.

## 6. UI work

`docs/design/fitforge-prototype.html` is the source visual reference — open it in a browser
and press `A` to toggle the annotations. Copy the screens your feature implements into
`specs/NNN-name/screenshots/`, and grade against those with the Visual Compliance Loop
(`docs/sdlc/review-process.md`). Never invent a layout when a reference exists; if the
reference is wrong, change the reference in the same feature and say why.

## 7. Governance changes ride alone

A change to the constitution, `docs/sdlc/`, or a rulebook goes on its own `docs/` branch and
merges on its own. Never bundle a rule change with the feature that made you want it — that is
how a rule gets adopted without anyone reviewing it as a rule.
