# Feature Specification: Identity and Member Profile

**Feature Branch**: `002-identity-member-profile`
**Created**: 2026-09-10
**Status**: Approved 2026-09-10 (owner: anas.m)
**Delivery Level**: Critical
**Input**: User description: "Registration, sign in, BFF-held session, and the member profile preferences"

## Why Critical

`docs/sdlc/critical-delivery.md` ("When a feature MUST be Critical") names **authentication
and authorization** outright, and this feature is both. It also carries the product's first
destructive operation (account deletion, invariant 10) and establishes the member scoping that
invariant 2 calls "a defect of the highest severity, not a bug to schedule".

`docs/roadmap.md` listed this row as Standard. A roadmap row carries no authority and the level
is chosen per feature at creation, so this is a corrected expectation rather than an amendment
to anything approved. **What it costs, stated up front so it is not a surprise at phase 1**:

1. a filled rollback plan **before** phase 1 begins;
2. an item-by-item pass over `modules/training/training-invariants.md` in *both* the AI and the
   human review, recorded in each;
3. retained audit evidence in this directory — gate command and exit code, scope-check verdict,
   `git diff --stat`, both review checklists;
4. **every phase gate run live by a human.** `**Gate Certification**: ci-held` and gate batching
   are both forbidden here — `scripts/enforcement-pack.ps1` fails the branch on either. Feature
   001 was certified `ci-held`; that route is not available for 002;
5. independent approval — the reviewer is not the owner. FitForge has two developers, so Ahmad
   reviews and **the solo-developer substitute (second-model review + 24-hour cooling-off) does
   not apply**.

## Context

Feature 001 delivered a shell that renders and an API that answers `/health`. Nothing in the
product knows who is asking. Every feature after this one depends on that answer: invariant 2
requires every read path to filter by the authenticated member, and there is currently no
authenticated member to filter by.

This feature introduces the first two entities (`Member`, `Profile`), the first migration with
real data in it, and the session seam the whole product sits on: **the C# API authenticates; the
Next.js BFF holds the session cookie** (visual reference annotation A6, invariant 7). Browser
code never holds a credential and never calls the API directly.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - A visitor becomes a member and gets in (Priority: P1)

Someone opens FitForge for the first time, registers with an email and a password, and lands in
the authenticated app shell. On a later visit they sign in and land in the same place.

**Why this priority**: nothing else in the roadmap can be built or demonstrated without it. It is
also the whole of INV-001.

**Independent Test**: register a new email, close the browser, reopen it, sign in, and observe the
authenticated shell — with no other feature present.

**Acceptance Scenarios**:

1. **Given** an unregistered email, **When** the visitor registers with a valid password, **Then**
   a member exists, a session is established, and the authenticated shell renders.
2. **Given** a registered email, **When** the member signs in with the correct password, **Then** a
   session is established.
3. **Given** a registered email, **When** the password is wrong, **Then** the same error appears as
   for an unknown email — "Email or password is incorrect." — and no session is established.
4. **Given** an email already registered in a different letter case, **When** someone registers with
   it, **Then** registration is refused as a duplicate.
5. **Given** an authenticated member, **When** they sign out, **Then** the session is destroyed
   server-side and the cookie is cleared; pressing Back does not restore an authenticated view.

### User Story 2 - The session lives in the BFF, not the browser (Priority: P1)

A member is signed in. Nothing the browser can read identifies them to the API; every call the page
makes goes to a BFF route handler, which attaches the credential server-side.

**Why this priority**: same priority as story 1 because it is not separable from it — a session
implemented the wrong way is not a later refactor, it is a rewrite of every route added after it.
This is the seam `docs/product/fitforge-logic.md` §6 and invariant 7 exist to protect.

**Independent Test**: sign in, then inspect `localStorage`, `sessionStorage` and every non-HttpOnly
cookie for anything that authenticates; and confirm the browser makes no request to the API origin.

**Acceptance Scenarios**:

1. **Given** an authenticated session, **When** browser storage is inspected, **Then** no token,
   password, or member identifier usable against the API is present.
2. **Given** an authenticated session, **When** the page loads any member data, **Then** the request
   goes to a BFF route handler and the API is called server-side.
3. **Given** no session, **When** an authenticated-only route is requested, **Then** the visitor is
   redirected to sign-in and no member data is rendered.
4. **Given** a session for member A, **When** any request supplies member B's `PublicId`, **Then**
   the API refuses — it never returns another member's data (invariant 2).

### User Story 3 - A member sets how the app talks to them (Priority: P2)

A member opens Profile and sets units, goal, experience level and time zone. Units change what they
see immediately; nothing stored changes.

**Why this priority**: P2 because the product works without it — the defaults are usable. It is
INV-011 and it is where invariant 4 (canonical storage, conversion at the edge) is established
before any weight is ever logged.

**Independent Test**: change each preference, reload, and confirm it persisted; switch units and
confirm displayed values change while the stored value does not.

**Acceptance Scenarios**:

1. **Given** an authenticated member, **When** they change units from `kg / cm` to `lb / in`,
   **Then** every displayed measurement re-renders converted and the persisted value is unchanged.
2. **Given** an authenticated member, **When** they change goal, experience or time zone, **Then**
   the change persists across a reload.
3. **Given** a time zone is set, **When** any day or week boundary is computed, **Then** it is
   computed in that zone.

### User Story 4 - A member changes their password or leaves (Priority: P2)

A member changes their password, confirming the current one; or deletes their account, which
soft-deletes them and their data and stops working immediately.

**Why this priority**: P2 for change-password. Account deletion is P2 only because no member has
data worth deleting yet — invariant 10 makes it non-optional before the product is real, and
shipping the control while it does nothing would be a UI that lies.

**Independent Test**: change the password and confirm the old one no longer works and other sessions
are gone; delete an account and confirm sign-in is refused afterwards.

**Acceptance Scenarios**:

1. **Given** an authenticated member, **When** they change their password with the correct current
   password, **Then** the new password works, the old one does not, and every other session for that
   member is invalidated.
2. **Given** an authenticated member, **When** the current password supplied is wrong, **Then** the
   change is refused and the existing password still works.
3. **Given** an authenticated member, **When** they delete their account, **Then** they are signed
   out, the member and their owned data are soft-deleted, and sign-in with those credentials is
   refused.
4. **Given** a member soft-deleted more than the retention window ago, **When** the retention process
   runs, **Then** the member and their data are permanently removed (invariant 10).

### Edge Cases

- Registration with an email that differs only in case, or by surrounding whitespace.
- Registration or sign-in while already holding a valid session.
- A session cookie that is present but references a session that has expired, been revoked, or
  belongs to a soft-deleted member.
- Repeated failed sign-in attempts against one email, and against many emails from one source.
- A time zone identifier the host platform does not recognise.
- The API being unreachable during sign-in — the member must be told the service is unavailable,
  never that their credentials are wrong.
- Password change submitted twice concurrently.
- Account deletion submitted by a member whose session was already revoked.

## Visual Inventory *(mandatory)*

Transcribed from `docs/design/fitforge-prototype.html` screens `signin` and `profile` — the
project prototype (source-of-truth rung 2) — and captured into this feature's `screenshots/`
(rung 1). Sizes below are taken from the prototype markup, not measured off the images.

**Capture conditions, stated because they bound what the images prove**: viewport 1280 CSS px
(the capture host's screen width — the tool could not resize below it), annotations enabled, page
zoom 0.8 for sign-in and 0.62 for profile so each screen fits one frame. **Apparent pixel sizes in
the images are therefore scaled; the VI numbers below are authoritative.** No capture exists below
1024px, so every responsive rule is specified here as a checkable fact instead (VI-001, VI-017) and
must be verified live in the Visual Compliance Loop rather than against an image.

### Screenshots: `screenshots/01-signin-desktop-light.jpg`, `screenshots/01-signin-desktop-dark.jpg` — sign in, 1280px

- **VI-001**: Grid, two columns at ≥1024px sized `1fr` and `380px`, gap 24px. **Below 1024px the
  left panel is not rendered** — the card alone, centered (annotation A1).
- **VI-002**: Left panel: rounded corners, 1px border in the border color, card background, padding
  40px; contents distributed top / middle / bottom over the full height.
- **VI-003**: Panel top: the FitForge dumbbell mark, 28×28, stroked in the primary color, beside the
  wordmark "FitForge" at 20px semibold with tight tracking in the foreground color; gap 8px.
- **VI-004**: Panel middle: headline "Log the set. Watch the number move." at 24px semibold, snug
  leading, tight tracking, max width 24rem; below it at 12px gap, body copy "Programs, an exercise
  library that knows what gear you have, and progress that comes from what you actually lifted." at
  14px in the muted color, same max width.
- **VI-005**: Panel bottom: "No nutrition tracking. No feed. Just training." at 12px muted.
- **VI-006**: Card: rounded corners, 1px border, card background, padding 24px.
- **VI-007**: Segmented control at the top of the card, 20px above the first label: muted
  background, 4px inset padding, rounded, two equal-width buttons "Sign in" and "Register".
  Selected = card background + small shadow + medium weight; unselected = muted foreground.
  **Sign in is selected by default** (A2).
- **VI-008**: Field order is Email then Password (A3). Labels 14px medium, 6px above their input.
- **VI-009**: Email input: height 40px, full width, rounded, input border, background color, 12px
  horizontal padding, 14px text, 2px focus ring in the ring color. 16px below it.
- **VI-010**: The password label sits on a row with "Forgot?" right-aligned at 12px muted, underline
  on hover (A3).
- **VI-011**: Password input identical to VI-009; 8px below it.
- **VI-012**: Error text renders **directly under the password field and above the submit button**,
  12px, destructive color, hidden when there is no error (A4). Copy: "Email or password is
  incorrect."
- **VI-013**: Primary submit: **full width, 40px tall** (A5), rounded, primary background,
  primary-foreground text, 14px medium. Label "Sign in".
- **VI-014**: Footer 16px below the button, centered, 12px muted: "New here? " followed by the link
  "Create an account" in the foreground color, underlined.
- **VI-015**: **No app shell on this screen** — no sidebar, no bottom bar, no header.
- **VI-016**: Both themes are references. Light and dark differ only in token values; no element
  appears, moves, or changes size between them.

### Screenshots: `screenshots/11-profile-desktop-light.jpg`, `screenshots/11-profile-desktop-dark.jpg` — profile, 1280px

- **VI-017**: Grid, gap 16px, **two equal columns at ≥1024px**; single column below. Left column is
  the Preferences card; the right column stacks its cards with 16px between.
- **VI-018**: Every card: rounded corners, 1px border, card background, padding 20px; heading 14px
  semibold.
- **VI-019**: Units: label "Units", then a **width-to-content** segmented control (muted background,
  4px inset, rounded) with "kg / cm" and "lb / in", 16px horizontal padding, 6px vertical.
  Selected = card background + shadow + medium weight.
- **VI-020**: Goal: label, then a 40px-tall full-width select. Options in this order: Hypertrophy,
  Strength, Endurance, General fitness.
- **VI-021**: Experience: label, then a 40px select. Options in this order: Intermediate, Beginner,
  Advanced.
- **VI-022**: Time zone: label, then a 40px select holding an IANA name with its UTC offset in
  parentheses, e.g. "Asia/Kuala_Lumpur (UTC+8)".
- **VI-023**: 8px under the time-zone select, 12px muted: "Weeks and streaks are counted in this
  zone." **This copy is mandatory** (K3) — it is the explanation a member needs when a streak breaks
  at an unexpected hour.
- **VI-024**: Account card: heading "Account", then a two-row list at 14px with 8px between rows,
  label muted on the left and value on the right: "Email" and "Member since" (date format
  `19 Aug 2026`).
- **VI-025**: 16px below, a wrapping row of 36px-tall buttons, 8px apart, ordered left to right:
  "Change password", then "Delete account" last.
- **VI-026**: "Delete account" is **the only destructive-colored control in the product** —
  destructive border and text, tinted destructive hover — and **always sits last** (K4).
- **VI-027**: 12px below the buttons, 12px muted: "Deleting soft-deletes your account and training
  data; it is recoverable for 30 days, then permanently removed (invariant 10)."
- **VI-028**: Changing units re-renders every displayed value immediately, and **no stored value
  changes** (K1, invariant 4).

### Declared deviations from the reference

Recorded here so they are decided, not discovered in the Visual Compliance Loop:

- **The "My gear" card is not built by this feature.** Gear is the exercise library's catalog
  (INV-003 / INV-012, owner ahmad, feature 003). Building it here would claim territory 003 needs
  and create a catalog neither feature owns. The right column holds the Account card alone until
  003 lands.
- **"Export my data" is not built by this feature** and its button is not rendered — hence VI-025
  lists two buttons where the reference shows three. No spec defines what the export contains or
  what format it takes, and a button that produces nothing is worse than an absent one.
- **"Forgot?" does not start a reset flow.** Password reset needs email delivery, which FitForge
  has no infrastructure for and no spec covering. The link is not rendered in 002; VI-010's row
  layout then applies to the label alone. This is the one deviation likely to surprise a member,
  and it is the first candidate for its own feature.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: A visitor MUST be able to register with an email address and a password. The email is
  the login identity and MUST be unique case-insensitively and after trimming surrounding
  whitespace.
- **FR-002**: The password policy MUST be stated in `plan.md` and enforced server-side, with a
  minimum length of at least 10 characters. Composition rules (mixed case, symbols) MUST NOT be
  imposed — length is the requirement that survives contact with real users.
- **FR-003**: Passwords MUST NOT be stored, logged, or transmitted anywhere in a recoverable form.
  Verification MUST use a salted, deliberately slow one-way hash; the algorithm and its parameters
  are a `plan.md` decision and MUST be recorded there.
- **FR-004**: Sign-in with correct credentials MUST establish a session. Sign-in with an unknown
  email and sign-in with a wrong password MUST be **indistinguishable** to the caller — same
  message (VI-012), same status, and no timing difference that identifies which one occurred.
- **FR-005**: The session MUST be held by the BFF in an `HttpOnly`, `Secure`, `SameSite` cookie.
  Browser-executed code MUST NOT be able to read any credential, and the browser MUST NOT call the
  API directly (invariant 7, annotation A6).
- **FR-006**: Signing out MUST destroy the session server-side, not merely clear the cookie.
- **FR-007**: Every authenticated-only route MUST redirect an unauthenticated visitor to sign-in
  without rendering member data. The sign-in screen MUST render with no app shell (VI-015).
- **FR-008**: Every API path that reads or writes member-owned data MUST be scoped to the
  authenticated member. There MUST be no parameter, header, or identifier by which one member's
  request can return another member's data (invariant 2).
- **FR-009**: `Member` and `Profile` MUST carry the project PK standard (`Id` + `PublicId`) and the
  audit fields. The internal `Id` MUST NOT appear in any URL, payload, or log line (invariant 8).
- **FR-010**: An authenticated member MUST be able to read and update unit preference, goal,
  experience level and time zone, and the change MUST persist.
- **FR-011**: Unit preference MUST be a display setting only. Stored values remain canonical
  (kg / cm) and MUST NOT be rewritten when the preference changes (invariant 4).
- **FR-012**: Time zone MUST be stored as an IANA identifier and MUST be the basis for every day and
  week boundary the product computes.
- **FR-013**: A member MUST be able to change their password by supplying the current one. On
  success, every other session for that member MUST be invalidated.
- **FR-014**: A member MUST be able to delete their account. Deletion MUST sign them out
  immediately, soft-delete the member and the data they own, and refuse subsequent sign-in.
- **FR-015**: A soft-deleted member and their data MUST be permanently removed after the retention
  window stated to the member (30 days, VI-027) — invariant 10's second half, which a soft delete
  alone does not satisfy.
- **FR-016**: Repeated failed sign-in attempts MUST be throttled, per email and per source, and the
  throttle MUST NOT reveal whether the email exists.
- **FR-017**: When the API is unreachable, sign-in MUST report a service problem and MUST NOT report
  a credential problem.
- **FR-018**: The API contract for every endpoint this feature adds MUST exist in
  `specs/002-identity-member-profile/contracts/` and be approved **before** either side implements
  against it (constitution VII).

### Key Entities

- **Member** — the login identity and the account. Email (unique, case-insensitive), password hash,
  display name, unit preference, time zone, experience level, goal, soft-delete marker and the date
  it was set, plus `Id` / `PublicId` and audit fields.
- **Profile** — slow-changing descriptive data belonging to one member: birth year, sex if given,
  height. Body weight is deliberately **not** here; it is a time series (`BodyMetric`) and arrives
  with a later feature.
- **Session** — whatever the BFF's cookie references and the API can revoke. Its shape is a
  `plan.md` decision; the requirement is that FR-006 and FR-013 can revoke it server-side, which
  rules out a self-contained token with no revocation path.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A visitor with no account reaches the authenticated shell in one screen and one
  submission, and a returning member in the same.
- **SC-002**: An automated test proves member A cannot read or write member B's profile through any
  exposed identifier. The test exists and fails when the scoping is removed.
- **SC-003**: After sign-in, no token, password, or API-usable identifier is present in
  `localStorage`, `sessionStorage`, or any script-readable cookie — verified by inspection, and the
  browser issues no request to the API origin.
- **SC-004**: A search of both repositories finds no plaintext or reversibly-encoded password in
  code, tests, logs, or fixtures.
- **SC-005**: The Visual Compliance Loop deviation table is empty at merge, except the three
  deviations declared above.
- **SC-006**: Both reviews contain an item-by-item pass over all ten training invariants (Critical
  addendum item 2), naming for each whether this feature touches it and how it is upheld.
- **SC-007**: Every phase's gate was run by a human and its exit code recorded in this directory
  (Critical addendum items 3 and 4).

## Assumptions

- **No email delivery exists**, so no flow in 002 sends mail: no verification email, no password
  reset. Registration therefore trusts the address as an identifier without proving control of it.
  This is acceptable for a product with no cross-member surface and no payment, and it is the
  assumption most likely to need revisiting — the first feature that emails a member also owes
  address verification.
- The retention window is **30 days**, because the visual reference states 30 days to the member
  (VI-027). If the number changes, both change together.
- Seeded gear, exercises and programs do not exist yet, so nothing in the profile depends on them;
  this is what makes the gear card cleanly severable.
- FitForge is not, today, operating under a regulation that dictates password or retention policy.
  The Critical level here follows from the kit's own criteria, not from an external auditor.
