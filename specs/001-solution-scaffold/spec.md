# Feature Specification: Solution Scaffold

**Feature Branch**: `001-solution-scaffold`
**Created**: 2026-09-10
**Status**: Approved 2026-09-10 (owner: anas.m)
**Delivery Level**: Standard
**Input**: User description: "Scaffold both repositories to a gate-green baseline and record the architecture ADR"

## Context

Adoption step 3 ("Define and PROVE the gate") already produced a *tool-default* scaffold in
each code repository — `dotnet new webapi` in **fitforge-api**, `create-next-app` in
**fitforge-web** — purely to prove each gate command can exit 0. That is a toolchain
receipt, not FitForge's baseline: neither repository yet has a layering, a design-token
system, an error contract, or a seam between the BFF and the API.

This feature turns those two default scaffolds into the baseline every later feature builds
on, and rehearses the whole ritual (branch → spec → plan → phase → gate → scope check →
cross-review → merge) while nothing of value is at stake. Per the bootstrap clause of
constitution principle IV, this feature's `plan.md` **is** the architecture until it merges.

No domain behaviour ships here. No exercise, program, session or member entity is created.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - A developer sees the two halves talking (Priority: P1)

A developer clones the nested layout, starts the API and the web app, and opens the site.
The app shell renders in FitForge's own design tokens — not Next.js starter chrome — and
tells them truthfully whether the C# API behind it is reachable, having asked it through
the BFF rather than from the browser.

**Why this priority**: it is the only story that proves the whole seam — browser → BFF route
handler → HTTP → C# API → back — in one observation. Every later feature is a variation on
that path, so if it is wrong, it is wrong everywhere. It also fixes the BFF boundary in
running code on day one, while the boundary is still cheap to place.

**Independent Test**: start both processes, load the site, observe the shell in FitForge
tokens and a reachable status; stop the API, reload, observe an unreachable status and no
unhandled error page.

**Acceptance Scenarios**:

1. **Given** both processes are running, **When** the developer loads the site root, **Then**
   the app shell renders with FitForge's palette and typography, and the health indicator
   reports the API reachable.
2. **Given** the API process is stopped, **When** the developer reloads, **Then** the health
   indicator reports the API unreachable, the page still renders, and no stack trace or
   unhandled exception reaches the browser.
3. **Given** the browser devtools network tab is open, **When** the page fetches health,
   **Then** the request goes to the web application's own origin — the browser never calls
   the C# API directly.
4. **Given** the API is asked for a route that does not exist, **When** it responds, **Then**
   the body is an RFC 9457 problem document, not framework default HTML.

---

### User Story 2 - A developer knows where new code goes (Priority: P1)

A developer starting feature 002 opens **fitforge-api** and can tell, without asking anyone,
where an entity goes, where a rule goes, where EF Core configuration goes, and which
direction dependencies are allowed to point — because the structure exists and one worked
example walks the entire path.

**Why this priority**: the decision is unavoidable and it is cheapest now. Deferring it means
feature 002 invents a layering under delivery pressure and every later feature inherits it by
accident. Constitution IV's bootstrap clause exists precisely so this decision is recorded in
a reviewed plan rather than discovered in a diff.

**Independent Test**: read `plan.md`'s ADR, then confirm each project it names exists in the
solution, that the project references point only the direction the ADR permits, and that the
health path is implemented through those layers rather than around them.

**Acceptance Scenarios**:

1. **Given** the solution, **When** a developer inspects project references, **Then** every
   reference points in the direction the ADR declares, and none points against it.
2. **Given** the ADR names a persistence boundary, **When** a developer searches the solution
   for database access outside it, **Then** there is none.
3. **Given** a new developer reads only `plan.md` and the rulebooks, **When** they are asked
   where a domain rule belongs, **Then** the answer is unambiguous.

---

### User Story 3 - The framework proves itself on every push (Priority: P2)

Both developers push to their own branches all week. On every push, CI runs each repository's
gate and grades that repository's phase commits against the feature's declared Territory —
so a scope violation or a broken build is caught by the machine, not by whoever happens to
review carefully.

**Why this priority**: the checks already exist and are already green; this story is about
making them *binding* rather than advisory, which is the difference between a framework and a
folder of documents. It is P2 only because the first two stories must exist for it to have
anything to grade.

**Independent Test**: push a commit that touches a path outside the declared Territory and
observe the code-repository scope check fail; correct it and observe it pass.

**Acceptance Scenarios**:

1. **Given** a phase commit inside the declared Territory, **When** CI runs, **Then** the
   scope check passes.
2. **Given** a phase commit touching a path outside it, **When** CI runs, **Then** the scope
   check fails with a non-zero exit code and names the offending path.
3. **Given** a push to the governance repository, **When** CI runs, **Then**
   `scripts/ritual-checks.ps1` runs every member and reports one verdict.

---

### Edge Cases

- **API unreachable at render time** — the shell must render and degrade, never blank or
  throw. Covered by US1 scenario 2.
- **API reachable but unhealthy** (process up, dependency down) — reported distinctly from
  unreachable; "up" and "healthy" are not the same claim.
- **Theme before hydration** — the viewer's chosen theme must be applied before first paint,
  or the page flashes the wrong theme on every load.
- **Viewport below 1024px** — the sidebar is not merely hidden; it is *replaced* by the
  bottom tab bar the prototype specifies (annotation B1).
- **No database configured** — the developer's first run must fail with a legible message
  naming the missing configuration, not an unhandled provider exception.

## Visual Inventory *(mandatory when the feature has `screenshots/`)*

Reference: `screenshots/fitforge-prototype.html`, screen `today` (`02 Today`) — the shell
chrome only. The dashboard content inside the shell (next-session card, stat tiles) belongs
to a later feature and is out of scope here; its annotations are noted so the shell is not
built in a way that makes them impossible.

### Screenshot: `screenshots/fitforge-prototype.html` — screen `today`, shell chrome, ≥1024px

- **VI-001**: Layout is a two-column grid, sidebar column fixed at 220px, content column
  fluid, 1rem gap (`lg:grid-cols-[220px_1fr]`).
- **VI-002**: Sidebar is a bordered card — 1px `--border`, `--card` background, radius
  `--radius`, 0.75rem padding.
- **VI-003**: Sidebar navigation order is fixed and never reordered: Today, Library,
  Programs, History, Progress. Profile & settings sits below a 1px divider with top margin,
  separated from the five.
- **VI-004**: The active navigation item uses `--secondary` background with
  `--secondary-foreground` text and medium weight; inactive items are `--muted-foreground`
  with an `--accent` background on hover only.
- **VI-005**: Navigation items are 0.75rem × 0.5rem padded, radius `md`, 0.875rem text,
  0.625rem gap between icon and label.
- **VI-006**: Below 1024px the sidebar is replaced by a bottom tab bar of exactly five items
  (the five above); "Profile & settings" moves into the header avatar. It is a replacement,
  not a hidden element (annotation B1).
- **VI-007**: The header is sticky to the top, 1px bottom border, `--card` background at 95%
  opacity with a backdrop blur.
- **VI-008**: Buttons are 40px tall (`h-10`). Primary is `--primary` fill with
  `--primary-foreground` text; secondary is a bordered transparent button with an `--accent`
  hover. Radius `md`.
- **VI-009**: Numeric display text uses the tabular-figure treatment the prototype's `.num`
  class applies — digits align in columns wherever they stack.
- **VI-010**: Typography is Inter, loaded as a webfont, with a system-UI fallback stack.
- **VI-011**: Light theme tokens exactly as declared on `:root` in the reference, including
  `--primary: 22 92% 50%` and `--radius: 0.65rem`.
- **VI-012**: Dark theme tokens exactly as declared for the dark class in the reference,
  including `--primary: 22 92% 54%`. Dark is a distinct token set, never a filter or an
  inversion of light.
- **VI-013**: Both themes are reachable and the choice survives a reload.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The API MUST expose a health endpoint reporting liveness and readiness as
  distinct facts.
- **FR-002**: The API MUST return errors as RFC 9457 problem documents for every unhandled
  failure and every validation failure, with no framework default error page reachable.
- **FR-003**: The API MUST be organised into the layers the ADR declares, with project
  references permitted in one direction only.
- **FR-004**: The persistence boundary the ADR names MUST be the only place where the
  database is configured, mapped and migrated; the pure domain layer MUST NOT reference an
  ORM, and raw SQL MUST NOT appear outside that boundary. This is constitution principle
  III's "the API owns the database" made structural.
- **FR-005**: The web application MUST implement the app shell to the Visual Inventory above,
  at both the ≥1024px and <1024px layouts.
- **FR-006**: The web application MUST carry FitForge's design tokens as the single source of
  colour, radius and typography; no component may hard-code a colour value.
- **FR-007**: The BFF MUST be the only caller of the C# API. The browser MUST NOT hold the
  API's address or call it directly.
- **FR-008**: The BFF MUST NOT open a database connection and MUST NOT hold any domain rule —
  restating the constitution's structural note where code review can enforce it.
- **FR-009**: The BFF↔API health contract MUST exist in `contracts/` and be agreed before
  either side implements it (constitution VII).
- **FR-010**: Configuration (API base address, secrets) MUST be read from environment, with
  `.env.example` carrying names and never values.
- **FR-011**: Each repository's gate command MUST exit 0 on the delivered baseline.
- **FR-012**: CI in each code repository MUST run that repository's gate and the cross-repo
  scope check on every push.
- **FR-013**: Both themes MUST be applied before first paint, with no flash of the wrong
  theme.

### Key Entities

None. This feature deliberately introduces no domain entity — the first entities arrive with
feature 002 (identity and member profile). Any persistence wiring delivered here is
structural only.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A developer who has never seen the repositories can go from clone to both
  processes running and the shell rendering by following `docs/onboarding.md` alone, with no
  question asked of another person.
- **SC-002**: Both gate commands exit 0 on the merged baseline, certified by the owner.
- **SC-003**: Every shell item VI-001 … VI-013 is satisfied or carries a recorded,
  user-approved deviation — the Visual Compliance Loop's deviation table is empty at merge.
- **SC-004**: Zero database calls exist outside the persistence boundary, and zero API calls
  originate from browser code — both verifiable by search.
- **SC-005**: A deliberately out-of-Territory commit fails the scope check, demonstrated once
  rather than assumed.
- **SC-006**: Feature 002 begins without needing to move, rename or re-layer anything
  delivered here.

## Assumptions

- The two default scaffolds already committed (fitforge-api `8bfd51b`, fitforge-web
  `159d8a0`) are the starting point; this feature reshapes them rather than starting over.
- .NET 10 and Node 22 are the pinned runtimes, as recorded in the Stack Profile.
- SQL Server is the database. No instance needs to exist for this feature to be gate-green:
  persistence wiring delivered here is structural, and no migration is applied.
- Authentication is out of scope. The shell may show a placeholder identity; no sign-in flow
  ships here (that is feature 002).
- The prototype at `docs/design/fitforge-prototype.html` is the authoritative visual
  reference, frozen for this feature into `screenshots/`.
- Deployment, containerisation and environment provisioning are out of scope.

## Out of Scope

- Any domain entity, migration applied to a real database, or seeded data.
- Sign-in, registration, session issuance (feature 002).
- Any screen beyond the shell — Today's content, Library, Programs, History, Progress.
- Observability beyond what the health endpoint needs.
