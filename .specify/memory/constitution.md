<!--
SYNC IMPACT REPORT
==================
Version change: kit template 0.6.0 -> FitForge 1.0.0 (first ratification) -> 1.1.0
(2026-09-10: Principle I gains the Amendment authority clause)
Bump rationale: MAJOR — initial ratification. This is FitForge's constitution, adopted
  from the Agentic SDLC Kit at kit version 0.6.0 (kit commit 2f187c9, which includes
  feature 012's cross-repository scope-check reach — the reason FitForge could be adopted
  as a multi-repository project with gate 4 actually enforced in the code repositories).
  Ratified by the owner on 2026-09-09.

  Slots resolved at ratification:
  - Principle II rung 1 (feature visual references): kept — FitForge has an authoritative
    visual reference, docs/design/fitforge-prototype.html, from which each feature copies
    the screens it implements into specs/NNN-name/screenshots/.
  - Principle II rung 2 (project-wide UI guidelines): the prototype itself. A feature's own
    screenshots outrank it, so a deliberate per-feature deviation is possible but must be
    written down in that feature's spec.
  - Principle III (Repository Separation): kept. Governance repository `fitforge`, backend
    `fitforge-api` (ASP.NET Core Web API — owns the domain and the database), frontend
    `fitforge-web` (Next.js App Router, including the BFF route handlers).
  - Principle V (Domain Invariants): modules/training/training-invariants.md.

  Structural note carried forward from the kit template: the BFF (Next.js route handlers)
  is NOT a second place where domain rules may live. It holds the session and aggregates
  calls to fitforge-api; it MUST NOT open a database connection or own an invariant. This
  is stated as a MUST NOT in the frontend and integration rulebooks because it is the
  boundary most likely to rot under delivery pressure.

Principles defined (10):
  I.    Specification First
  II.   Source of Truth Hierarchy
  III.  Repository Separation
  IV.   Architecture Consistency
  V.    Domain Invariants                (modules/training/training-invariants.md)
  VI.   Security
  VII.  External Integration Governance
  VIII. Testing Requirements
  IX.   Human Review Requirement
  X.    Controlled Delivery

Templates requiring updates when this file changes:
  - .specify/templates/plan-template.md (Constitution Check gate must mirror the principles 1:1)
  - .specify/templates/tasks-template.md (test-policy language must not contradict Principle VIII)
  - CLAUDE.md (strict rules must not contradict this file)
  - scripts/enforcement-pack.ps1 (encodes constitutional constants — batch-phase cap,
    Critical cooling-off hours, the Gate Certification legal values `user-run`/`ci-held`
    and the Critical ci-held exclusion, the Micro-lane bounds (territory-file cap 5,
    phase-line hard bound 400, single-phase rule) and the Delivery Level legal values —
    these MUST change in lockstep with amendments touching them)

Amendments received from the kit after ratification are recorded below by re-expression
(adoption/updating.md), never by copying the kit's own version history back in.
-->

# FitForge Constitution

## Core Principles

### I. Specification First

All work MUST begin with specification and planning before implementation. The required
workflow is: (1) create or update `spec.md`; (2) create or update `plan.md`; (3) create or
update `tasks.md`; (4) implement one approved phase only; (5) run the project gate; (6) review
changes; (7) commit the approved phase. Implementation MUST NOT start before requirements are
documented.

**Micro arm**: a feature declared **Micro** (Principle X, Micro lane) satisfies this
principle with an approved **single-page mini-spec** — its `spec.md`, authored from
`.specify/templates/micro-spec-template.md` — alone: steps (2) and (3) are skipped, and no
`plan.md` or `tasks.md` exists while the feature remains Micro. Every other step is
unchanged. Absent a Micro declaration, the full workflow above applies.

**Rationale**: Documented intent prevents rework, makes review meaningful, and ties every code
change to an approved requirement. The Micro arm keeps all of that — intent is still written
and approved before implementation — while dropping only the planning ceremony that adds
nothing to a change small enough to fit the lane's machine-policed bounds.

**Amendment authority**: once a feature's `spec.md` or `plan.md` has been approved, any
later change to that feature's `spec.md`, `plan.md`, `tasks.md` or `contracts/` — a new
package, a changed value, an added phase, a widened Territory, a reinterpreted contract
clause — MUST record who approved it. The amended section carries an
`**Amendment approved by**: <name>, <YYYY-MM-DD>` line, and the amendment commit names the
same approver. **An implementing agent MUST NOT approve its own amendment.** Amending
before implementing satisfies the sequence; it does not satisfy this rule.

**Rationale**: added 2026-09-10 after the AI review of feature 001 (governance F3) found
that every amendment following that feature's single owner approval had been written by
the implementing session and implemented against minutes later — twenty-nine seconds, in
one case — with no approver anywhere. No machine check caught it, because
`scope-check`, `scope-check-repos`, `enforcement-pack` and `doc-lint` grade paths, tokens
and dates, and none of them grades authority. Every one of those amendments happened to be
correct, which is exactly why the mechanism would have survived one that was not. Getting
the order right — amend, then implement — is a check on retroactivity, not a check on
consent, and the two had been quietly conflated.

**Enforcement, honestly stated**: this clause is enforced by review, not by machine. The
natural home for the check is `scripts/enforcement-pack.ps1`, which `kit-manifest.json`
classes `verbatim` — a rule added to it here would be reverted by the next kit update,
leaving a constitutional requirement whose check had quietly disappeared. That is worse
than no check, because nobody would notice. The machine check is therefore owed to the kit
as its own feature; until it exists, the AI review and the human reviewer are what stands
between this rule and the habit it was written to break.

### II. Source of Truth Hierarchy

The following order of precedence MUST always be respected:

1. Feature visual references (screenshots / prototype captures) — *include this rung only if
   the project has an authoritative visual reference; otherwise delete it*
2. `docs/design/fitforge-prototype.html` — the project-wide UI reference
3. `spec.md`
4. `plan.md`
5. API contracts
6. Data model
7. `tasks.md`
8. Research documents
9. Notes

If a higher rung and a lower rung conflict, implementation MUST stop and the conflict MUST be
reported. Lower-fidelity artifacts MUST NOT silently override higher-fidelity intent. When
visual references exist, new UI layouts MUST NOT be invented.

**Rationale**: A single, ordered source of truth removes ambiguity and prevents lower-fidelity
artifacts from silently overriding higher-fidelity intent.

### III. Repository Separation

<!-- OPTIONAL: delete this principle (and renumber) for single-repository projects. -->

FitForge uses separate repositories. The backend repository is `fitforge-api`. The
frontend repository is `fitforge-web`. Backend and frontend code MUST NOT be mixed in the
same repository unless explicitly approved in the technical plan.

**Rationale**: Separation keeps deployment, security boundaries, and ownership clean across
tiers.

### IV. Architecture Consistency

The existing architecture is the source of truth. New features MUST follow the existing
architecture. New architectural patterns, new frameworks, new UI libraries, and new persistence
approaches MUST NOT be introduced unless explicitly approved in the technical plan.

**Bootstrap clause**: at project creation there is no existing architecture to follow. During
the initial scaffold feature (the project's first numbered feature — see
`adoption/greenfield.md`, step 4), "the existing architecture" means the architecture selected
and approved in that feature's `plan.md`, which MUST record the decision ADR-style: options
considered, the decision, and its consequences. Once the scaffold feature is merged, that
architecture becomes the existing architecture and this principle applies in full.

**Rationale**: Consistency lowers maintenance cost and keeps the system reviewable by the whole
team. Without the bootstrap clause, "follow the existing architecture" is undefined on an empty
repository — the clause anchors the rule to an approved plan instead of leaving the agent to
improvise one.

### V. Domain Invariants

The non-negotiable rules of this project's domain are defined in
`modules/training/training-invariants.md` and carry constitutional force. Agents
and reviewers MUST treat a domain-invariant violation exactly like a violation of this file.

**Rationale**: Every serious domain has rules that must survive any refactor (immutability of
postings, consent trails, order-state machines). Naming them once, with constitutional force,
stops an agent from "creatively" violating them.

### VI. Security

Authentication is required for protected functionality. Authorization is required for protected
operations. Secrets MUST NEVER be stored in source code. Sensitive information MUST NOT be
logged. All external integrations MUST use secure authentication mechanisms.

**Rationale**: Security controls are non-negotiable architecture concerns, not cleanup tasks.

### VII. External Integration Governance

All external integrations require documented contracts. Each contract MUST define purpose,
authentication, endpoints, request schema, response schema, error schema, timeout policy, retry
policy, idempotency strategy, and audit requirements. Undocumented integrations are prohibited.

**Rationale**: Documented contracts make integrations testable, recoverable, and safe to change.

### VIII. Testing Requirements

Business-critical functionality requires automated tests. Business-critical calculations require
deterministic validation (golden fixtures where outputs must be exact). Changes affecting
business-critical logic require regression coverage.

**Rationale**: Deterministic, regression-covered tests are the only credible guarantee that
critical logic remains correct across changes — especially changes made by an AI agent.

### IX. Human Review Requirement

AI review alone is insufficient. Human review is required before merge. Human reviewers MUST
verify business requirements, domain correctness, security implications, visual-reference
compliance (where visual references exist), and architectural compliance.

**Rationale**: Business and architectural correctness require human accountability that
automated review cannot replace.

### X. Controlled Delivery

Work MUST be delivered incrementally. Only one approved phase MAY be implemented at a time.
Unrelated changes MUST NOT be included in the same feature implementation. Every completed phase
MUST pass project gates — run by the user, with the exit code confirmed by the user — before
proceeding. An AI agent MUST NOT claim success without that confirmation.

**Batched gates**: for a Lite or Standard feature, the approved plan MAY declare — before the
batch's first phase is implemented, as `**Gate Batching**: phases N-M` in `plan.md` — that a
run of at most **3 consecutive phases** shares one certifying user-run gate at the end of the
batch. Within a batch, every phase still requires its own commit, its own scope check, and its
own AI review, and the agent still runs the gate per phase for feedback; only the user-run
certification moves to batch end. Critical features MUST NOT declare batches — their per-phase
user-run gate obligation is unchanged. Absent a declaration, the per-phase user-run gate above
applies in full.

**CI-held certification**: for a Lite, Micro, or Standard feature, the approved plan MAY
declare — as `**Gate Certification**: ci-held` in `plan.md` (for a Micro feature: in its
approved mini-spec `spec.md`, the lane's only specification document), before the first
phase it governs — that
gate certification is satisfied by the owner's **recorded approval on the evidence triplet**:
the CI run of the project gate on the **exact phase commit** (for a declared batch, the
batch-end commit), cited by run URL, green conclusion, and commit sha, recorded in the
feature's phase record. The approval is per phase (or per declared batch), never blanket;
a run on any other commit certifies nothing; the agent's obligations are unchanged — it
MUST NOT claim success, and under this mode it reports the evidence and requests the
owner's approval on it. The user-run gate remains lawful always. Critical features MUST NOT
declare or use CI-held certification. Absent a declaration, the value is `user-run` — the
gate law above applies in full.

**Micro lane**: a numbered feature MAY be declared **Micro** — `**Delivery Level**: Micro`
in its `spec.md` — when it fits the lane's bounds. A Micro feature has **exactly one
phase**; its specification is the approved single-page mini-spec (Principle I, Micro arm),
which carries the feature-global **Territory** block and, optionally, a
`**Gate Certification**` declaration; it MUST NOT declare `**Gate Batching**` (one phase —
nothing to batch). The measurable bounds are hard: the declared Territory covers at most
**5 files** (the feature's own `specs/NNN-name/**` excluded), and the phase changes at
most **400 lines in total** — counted across every commit carrying its `phase N` token,
so remediation commits cannot split the bound — a failure for Micro where other lanes
get a per-commit warning. The
non-measurable bounds — no schema or migration, no new packages, no architecture change,
no domain-invariant surface, no visual-reference UI — are affirmed in the mini-spec's
eligibility checklist and verified in human review. Every verification layer is unchanged:
user-held gate certification, the machine scope check (Territory read from `spec.md`),
fresh-context AI review, and human review at merge. Critical features MUST NOT use the
Micro lane. Absent a `**Delivery Level**` declaration, a numbered feature is Standard.
When work outgrows any bound, the feature is **promoted in place to Standard**: `spec.md`
is expanded to the full template (level re-declared Standard) and `plan.md` + `tasks.md`
are added (Territory moves under the phase headings), in a commit made **before** any
further phase commit; promotion is one-way and all-or-nothing.
`scripts/enforcement-pack.ps1` fails a Micro branch that violates any of these bounds.

**Rationale**: Small, gated increments keep changes reviewable, reversible, and low-risk; the
user-held exit code keeps the trust boundary human. Batching trades gate frequency — never
per-phase revertibility or review — for fewer owner interruptions on low-risk work, and only
when declared in an approved plan. CI-held certification moves the owner's approval input
from "I ran it" to "I read unforgeable evidence" — the agent cannot mint a green run on a
host it does not control — without ever removing the human approval itself; the trust
boundary stays human, asynchronously. The Micro lane trades specification ceremony — never
verification — for speed on provably small work: every measurable bound is machine-policed,
and outgrowing a bound forces promotion to Standard rather than quiet stretching.

## Governance

This constitution supersedes all other development practices. When any rule, document, or
generated artifact conflicts with this constitution, the constitution prevails. This file is the
project's ONLY constitution — do not create a second copy elsewhere in the repository; other
documents may point here.

**Amendment procedure**: Amendments MUST be proposed as a documented change to this file,
including rationale and impact on dependent templates (`plan-template.md`, `spec-template.md`,
`tasks-template.md`) and runtime guidance (`CLAUDE.md`, `docs/`). An amendment is adopted only
after human approval. Update the SYNC IMPACT REPORT header with every amendment.

**Versioning policy**: This constitution is versioned using semantic versioning.
MAJOR — backward-incompatible governance or principle removals or redefinitions.
MINOR — a new principle or section is added, or guidance is materially expanded.
PATCH — clarifications, wording, and non-semantic refinements.

**Compliance review**: All specs, plans, tasks, pull requests, and reviews MUST verify
compliance with these principles. The Constitution Check gate in `plan-template.md` MUST be
evaluated before Phase 0 research and re-evaluated after Phase 1 design. Any violation MUST be
justified in the plan's Complexity Tracking section or the work MUST stop and be reported. Use
`CLAUDE.md` and the `docs/` guidance files for runtime development guidance.

**Version**: 1.1.0 | **Ratified**: 2026-09-09 | **Last Amended**: 2026-09-10
