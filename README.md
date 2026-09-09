# FitForge

A workout and exercise app for gym members: an exercise library that names the gear each
movement needs, training programs, logged sessions, and progress over time.

This is the **governance repository**. It holds the law, the specs, and the design
reference — no runnable source code. The code lives in two independent repositories cloned
inside this one:

```text
fitforge/                      # this repository — governance
├── fitforge-api/              # ASP.NET Core Web API — owns the domain and the database
└── fitforge-web/              # Next.js App Router — the web app and its BFF route handlers
```

Clone all three in that nested shape. A standalone clone of a code repository inherits no
governance — see `docs/sdlc/repository-strategy.md`.

## The product in one paragraph

A member signs in, sees today's planned workout, starts a session, logs each set (reps, load,
RPE) against a rest timer, and finishes. The app derives their volume, estimated 1RM, personal
records, streak and program adherence from those sets — never from stored summaries. Programs
are authored from the exercise library; the library records, per exercise, which muscle groups
it trains and which gear it needs. There is no store, no payments, no nutrition tracking and
no social feed.

Product detail: **docs/product/fitforge-logic.md**.
Rules that outrank every feature: **modules/training/training-invariants.md**.
Visual reference: **docs/design/fitforge-prototype.html** (open it in a browser).

## How work happens here

FitForge is delivered under the Agentic SDLC Kit: spec before code, one phase per commit,
territory declared before it is touched, and a gate whose success only the feature's owner may
declare. If you are an AI agent, `CLAUDE.md` is your entry point and
`.specify/memory/constitution.md` outranks it.

- The ritual on one page: `docs/sdlc/flow.md`
- What "done" means: `docs/sdlc/definition-of-done.md`
- What to build next: `docs/roadmap.md`
- Joining the project: `docs/onboarding.md`

Two developers, Anas and Ahmad, each own whole features end to end and review each other's.
Nobody reviews their own feature, and nobody merges without the other's review.

## Checking the project is healthy

```bash
pwsh -File scripts/ritual-checks.ps1
```

One command, run identically here and in CI. It is the only source of a verdict; a green
terminal in a chat transcript is not evidence.
