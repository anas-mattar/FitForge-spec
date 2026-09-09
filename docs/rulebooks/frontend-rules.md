# Frontend Rules — FitForge

> **Binding**: this rulebook is enforced through the frontend tier's compliance checklist
> (Definition of Done item 5, `docs/sdlc/definition-of-done.md`): the checklist asserts,
> this rulebook explains. Visual references (rung 1 of Source of Truth) outrank it: never
> invent a layout when screenshots exist, and UI phases with visual references run the
> Visual Compliance Loop (`docs/sdlc/review-process.md`).

## Structure

- Code layout: fixed ADR-style in **specs/001-solution-scaffold/plan.md** (constitution IV,
  bootstrap clause). Until that feature merges there is no existing layout to follow. This
  line is rewritten with the actual layout in the same feature that establishes it.
- shadcn/ui components are **vendored** into the repository, not imported from a package.
  A vendored component MAY be edited; the edit MUST be noted in the feature's `plan.md`.
  **Why**: that is how shadcn works — pretending it is a dependency produces upgrade rituals
  that do not exist.
- New routes, folders, or shared components follow the existing layout; new top-level
  structure requires approval in the feature's `plan.md`.

## Data Flow

- ALL calls to `fitforge-api` MUST go through the BFF route handlers under `app/api/**`,
  and every BFF call to the API MUST go through one typed client module. Browser code MUST
  NOT call `fitforge-api` directly, and MUST NOT hold an API credential.
  **Why**: one place for base URL, auth, timeouts, and error mapping — and the member's
  session token never reaches the browser.
- The BFF MUST NOT open a database connection, and MUST NOT enforce, duplicate, or work
  around any rule in **modules/training/training-invariants.md** (§7). It holds the session
  and aggregates; it decides nothing.
  **Why**: this is the boundary that rots first, and a second enforcement point means two
  answers to every domain question.
- Response types MUST mirror the feature's contract (the feature's `contracts/`
  directory) and cite it in a comment. **Why**: the citation makes rung-checking possible.
- Derived business values (totals, conversions, rounding) MUST come from the backend; the
  frontend renders them and MUST NOT re-compute them.
  **Why**: two implementations of one formula always diverge.

## UI States

- Every data surface MUST implement all states that apply: loading, empty,
  error/unavailable, and populated — and the error state MUST be visually and
  programmatically distinguishable from empty.
  **Why**: an error rendered as an empty list is silent data loss to the user.
- Every data surface MUST carry a `data-state` attribute naming the active state
  (`loading` / `empty` / `error` / `ready`), so tests assert the state rather than guessing
  from rendered text.

## Forms & Input

- All data entry MUST use React Hook Form with a `zodResolver`; hand-rolled `useState`
  field state is prohibited. **Why**: it is the difference between validation that exists
  and validation that existed when the form was written.
- The zod schema for a form MUST be the same schema the BFF route handler validates with.
  **Why**: a client schema that drifts from the server's is worse than no client schema —
  it teaches the member the wrong rules.
- Set logging (reps, load, RPE) MUST accept the decimal separator the member's locale
  produces and MUST submit a canonical value.
  **Why**: `12,5` silently becoming `125` kg is the most likely real data-corruption bug in
  this product.

## Styling

- Tailwind utility classes only; no separate CSS modules or styled-components.
- Colors MUST come from the shadcn CSS-variable tokens (`bg-background`, `text-foreground`,
  `border-border`, …) — never a raw hex value or a bare Tailwind palette color.
  **Why**: the token set is what makes light and dark correct by construction, and it is
  what **docs/design/fitforge-prototype.html** is drawn in, so implementation stays
  transcription rather than reinterpretation.
- Numeric columns (loads, volumes, dates in tables) MUST use `tabular-nums`.
- Every screen MUST be usable at 375px wide. **Why**: this is a phone app used standing at
  a rack; the desktop layout is the secondary case.

## Security

- Server-only secrets and env vars MUST NOT reach client-side code or be exposed through
  public-prefixed variables. **Why**: everything shipped to the browser is public.
- The member session MUST live in an httpOnly, secure cookie issued by the BFF; tokens
  MUST NOT be placed in `localStorage` or any client-readable store.
- A server component or BFF handler MUST NOT return another member's data to satisfy a
  layout convenience; scoping is enforced by `fitforge-api` and MUST NOT be re-implemented
  or relaxed here (**modules/training/training-invariants.md** §2).
