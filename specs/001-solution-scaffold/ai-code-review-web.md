# AI Code Review — 001 Solution Scaffold (fitforge-web, phases 3 and 4)

**Reviewer**: fresh-context agent — claude-opus-5
**Date**: 2026-09-10
**Branches**: fitforge-web `001-solution-scaffold` (tip `8d9232d`) — phase 3 `e0df410`, phase 4 `8d9232d`. Governance repo read at `ae7c38d`; `fitforge-api` not reviewed here.
**Scope reviewed**: every file in `fitforge-web/src/` at tip (`app/globals.css`, `app/layout.tsx`, `app/page.tsx`, `app/api/health/route.ts`, `components/shell/*` — 7 files, `components/ui/button.tsx`, `lib/{utils,units,health,api-client}.ts`, `lib/__tests__/{health,units}.test.ts`), both phase diffs in full, `package.json` / `package-lock.json` / `components.json` / `next.config.ts` / `eslint.config.mjs` / `tsconfig.json` / `.env.example`, the compiled CSS and JS under `.next/static`, the prerendered `.next/server/app/index.html`, and the running application under `next start`.
**Feature contract**: Standard delivery, `ci-held` certification, no `plan.md` amendment. Phase 3 Territory `fitforge-web/{src/**,package.json,package-lock.json,components.json,postcss.config.mjs,next.config.ts}`; phase 4 Territory `fitforge-web/{src/**,.env.example,package.json,package-lock.json}`. Packages limited to plan §5's fitforge-web list. No colour literal in any component (FR-006). The browser never holds the API address (FR-007). Both themes before first paint (FR-013).

## Reviewer Provenance

- **Reviewer**: fresh-context agent — claude-opus-5 (no implementation context; given only the diffs, spec, plan, tasks, contract, visual reference and rulebooks)
- **Implementer**: claude-opus-5 (the implementing session; its reasoning was NOT provided to this reviewer)
- **Inputs provided**: phase 3/4 diffs, full source and test tree, spec.md, plan.md, tasks.md, contracts/health.md, the prototype HTML, constitution, rulebooks
- **Attestation**: This reviewer did not produce the diff under review.

## Verdict

**REQUEST CHANGES** — Phase 3 transcribes the design tokens with genuine fidelity (all 39 light/dark tokens machine-compared, zero mismatches) and phase 4 gets the load-bearing architectural fact right: I built the app and confirmed independently that `FITFORGE_API_BASE_URL` and `health/ready` appear **nowhere** in `.next/static`, and I confirmed all four mapping-table rows end-to-end against the real route with a stub upstream. That part of the claim survives audit. What does not survive is the layer above it. `ApiHealthIndicator` feeds an unvalidated network value straight into an object index and destructures the result — a `/api/health` body the component does not expect crashes the root layout and takes the whole shell to the global error page, which is precisely what US1 scenario 2 forbids. A missing `FITFORGE_API_BASE_URL` is reported to the operator as `unreachable` with **zero** server-side trace, which makes the careful error message in `api-client.ts` unreachable in practice and sends a developer to debug the API process instead of their environment — the same wrong-process failure contract §2 exists to prevent. T040 and T041 were not implemented at all: the route handler the browser actually calls has no automated test of any kind. And the Visual Compliance Loop's recorded "PASS" on VI-004 is wrong — the implementation changes the inactive nav item's *text* colour on hover and the reference does not. The residual risk sits in the two client-side surfaces (the indicator and the theme toggle) and in a recorded visual result that I could not independently confirm because no screenshots were attached.

## What was verified (evidence)

| Area | Evidence |
|---|---|
| Spec match (FRs implemented as specified) | **FR-005/006**: `grep -rnE '#[0-9a-fA-F]{3,8}\|rgb\(\|rgba\('` over `src/` returns only the `hsl(var(--token))` bridges in `globals.css:67-103`; zero bare-palette classes (`bg-(slate\|zinc\|red\|…)-[0-9]` → NONE). **FR-007**: built with `npm run build`, then `grep -rl FITFORGE_API_BASE_URL .next/static` → 0 files; `grep -rl 'health/ready' .next/static` → 0 files; `grep -rlE 'probeApiHealth\|classifyStatus' .next/static` → 0 files. Both strings appear only in `.next/server/chunks/[root-of-the-server]__0m_cmf5._.js`. `api/health` appears in exactly one client chunk (`.next/static/chunks/05uc0mkq5htq5.js`) — this application's own origin, as required. **FR-008**: no DB driver in `package.json`, no `pg`/`mssql`/`prisma` import anywhere in `src/`. **FR-010**: `.env.example` carries `FITFORGE_API_BASE_URL=` and `FITFORGE_SESSION_SECRET=`, names only, no `NEXT_PUBLIC_` prefix. **FR-011**: `npm run lint` exit 0, `npm run typecheck` exit 0, `npm test` exit 0 (20 passed), `npm run build` exit 0 — agent-run feedback only, not certification. **FR-013**: read `.next/server/app/index.html` — the theme IIFE is emitted as a parser-blocking inline `<script>` at the end of `<head>`, after the stylesheet `<link>` and before `<body>`; it therefore runs after CSS is available and before first paint. Verified the emitted source is byte-identical to `THEME_SCRIPT`. |
| Visual-reference match (where references exist): Visual Compliance Loop deviation table attached, empty or user-approved (`docs/sdlc/review-process.md`) | **Not clean — see the deviation table below.** Tokens machine-compared: a script parsed `:root`/`.dark` out of `screenshots/fitforge-prototype.html` and out of `src/app/globals.css` and diffed them key-by-key → 20 light tokens, 0 mismatches; 19 dark tokens, 0 mismatches (VI-011 and VI-012 independently CONFIRMED, including `--primary: 22 92% 50%` / `22 92% 54%` and `--radius: 0.65rem`). Compiled CSS (`.next/static/chunks/34k481vretzqs.css`) confirms `.rounded-lg{border-radius:var(--radius)}`, `.rounded-md{border-radius:calc(var(--radius) - 2px)}`, `@media (min-width:64rem){.lg\:grid-cols-\[220px_1fr\]{grid-template-columns:220px 1fr}}`, `.backdrop-blur{--tw-backdrop-blur:blur(8px)}`, `.bg-card\/95{background-color:color-mix(in oklab, hsl(var(--card)) 95%, transparent)}`, `.num{font-variant-numeric:tabular-nums}` (byte-identical to the prototype's `.num` rule), and `.font-sans{font-family:var(--font-inter), ui-sans-serif, system-ui, sans-serif}` matching the prototype's `fontFamily.sans` array exactly, with 12 self-hosted `@font-face` Inter subsets. Two frozen copies of the prototype (`docs/design/` rung 2 and `screenshots/` rung 1) differ only in line endings — `diff <(tr -d '\r' …) <(tr -d '\r' …)` is empty. Three real deviations found that the recorded table marks PASS or omits (F5, F6, F7). |
| Feature contract held (no unapproved table/migration/permission/package) | `git show e0df410 -- package.json` adds `class-variance-authority`, `clsx`, `lucide-react`, `tailwind-merge`; `git show 8d9232d -- package.json` adds `server-only`. All five are on plan §5's approved fitforge-web list. `vitest ^5.0.0` predates the branch (present in `git show 159d8a0:package.json`) and is named in plan §Technical Context. `tailwindcss-animate` was approved but not installed — narrowing, lawful. No migration, table or permission exists in this repository. |
| Constitution / domain invariants | **II**: the prototype (rung 1/2) outranks spec and plan; three places diverge from it without a recorded deviation (F5, F6, F7) — reported, not silently reconciled. **IV**: no package outside plan §5; ADR §4.6's server-component default is honoured — only `NavLink`, `ThemeToggle` and `ApiHealthIndicator` carry `"use client"`, and the prerendered HTML shows `Header`, `Sidebar` and `BottomNav` fully server-rendered. **V**: `src/lib/units.ts` converts for display only and cites invariant §4; nothing in `src/` persists or re-derives a business value. **VI**: no secret in source; `.env.example` names only; the readiness path discloses nothing beyond three words — verified by curling the live route. **VII**: both `health.ts:2` and `route.ts:6` cite `contracts/health.md` by path and section, as integration-rules requires; `ApiHealthIndicator.tsx` does not (F4). **VIII**: the contract's mapping rules are covered; the route that applies them is not (F3). **X**: phase 4 reverts cleanly (`git show 8d9232d \| git apply -R --check` → clean). |
| Security (authn/authz, secrets, sensitive logging) | `server-only` enforcement is real, not decorative: `node_modules/server-only/package.json` resolves `"react-server"` → `empty.js` and `"default"` → `index.js`, which is a bare `throw new Error("This module cannot be imported from a Client Component module…")`. A client component importing `api-client.ts` therefore fails at module evaluation during the client build. Bundle grep above confirms no leak today. No secret is logged — but nothing is logged at all, which is itself a defect (F2). `/api/health` is unauthenticated and uncached with a 10 s upstream fan-out (F18). No `NEXT_PUBLIC_` variable exists. |
| Scope guard (`scope-check.ps1` PASS on the phase commit; `git diff --stat` read for intent) | `pwsh -File scripts/scope-check-repos.ps1` from `D:\solutions\fitforge`: `scope-repos: fitforge-api: PASS phase 6 commit cded3bd (3 file(s))` / `scope-repos: fitforge-web: PASS phase 4 commit 8d9232d (8 file(s))`, **exit code 0** (captured explicitly via `& {…; exit $LASTEXITCODE}`). Both diffs read in full: phase 3 = 15 files / +569 −97, phase 4 = 8 files / +322 −1, every path inside its phase's declared Territory. No unrelated file touched; the unused `public/*.svg` starter assets are correctly left alone as out-of-Territory. |
| Rollback safety (phase reverts cleanly; schema additive?) | `git show 8d9232d \| git apply -R --check` → clean, so phase 4 is independently revertible from the tip. Phase 3 does not revert in isolation (`Header.tsx`, `package.json`, `package-lock.json` conflict) because phase 4 edited the same files — expected, and LIFO revert works. No schema, no migration, no persisted state in this repository; the only client-side persistence is `localStorage["fitforge-theme"]`, which is read defensively inside `try/catch` on both the write (`ThemeToggle.tsx:37-41`) and the read (`theme-script.ts:17-23`), so a stale or absent value degrades to the OS preference rather than failing. |

### Visual Compliance Loop — this reviewer's deviation table

**I was not able to independently confirm the recorded result.** `tasks.md` records thirteen PASS verdicts measured from computed styles in Chrome at 1280px and 357px. No screenshots were attached, and `docs/sdlc/review-process.md` step 5 is explicit: *"Attach the final table and both screenshots to the phase notes — the AI review verifies they exist."* `specs/001-solution-scaffold/screenshots/` contains only `fitforge-prototype.html`. I therefore re-derived what is statically checkable (tokens, radii, grid, blur, `.num`, font stack, class-by-class comparison of the sidebar markup against the prototype's `today` screen) and could not re-derive the render-time measurements at all. Within what I could check, three rows of the recorded table are wrong or incomplete:

| # | Element (VI ref) | Reference shows | Implemented shows | Severity | Resolution |
|---|---|---|---|---|---|
| 1 | VI-004 — inactive nav hover (`prototype today` sidebar, `hover:bg-accent`) | Inactive item: `text-muted-foreground hover:bg-accent`. Background only changes on hover; the text stays `--muted-foreground`. | `NavLink.tsx:62` — `text-muted-foreground hover:bg-accent hover:text-accent-foreground`. Text also jumps to `--accent-foreground` (light `240 6% 10%` vs `240 4% 46%` — a visible darkening). Confirmed in the compiled CSS: `.hover\:text-accent-foreground:hover{color:hsl(var(--accent-foreground))}`. | Low visual, but it is an unrecorded divergence from rung 1 and VI-004's wording ("`--accent` background on hover **only**") | Recorded as **PASS** in `tasks.md`. Either remove `hover:text-accent-foreground` or record the row and get owner approval. |
| 2 | VI-003/VI-005 — nav item icons | The prototype's `today` sidebar items are plain text anchors with no icon at all (`<a class="flex items-center gap-2.5 rounded-md px-3 py-2 …">Today</a>`). | Six lucide icons invented and assigned: `CalendarCheck`, `Dumbbell`, `LayoutList`, `History`, `TrendingUp`, `Settings` (`navigation.ts:23-39`). | Medium — VI-005 does say "gap between icon and label", so *an* icon is sanctioned by the spec; *which* icons is not in any rung. | Not in the recorded table at all. Owner decides: accept the six choices explicitly, or drop them to match the reference. |
| 3 | VI-001 — content container | Prototype `<main class="mx-auto max-w-[1400px] px-4 py-6">` wrapping the `today` grid. | `layout.tsx:38` — `mx-auto grid w-full max-w-6xl … px-4 py-4` (72rem = 1152px), and the header inner is `max-w-6xl` too. At ≥1400px the content column is ~250px narrower than the reference, and vertical padding is 16px rather than 24px. | Low-medium, but it is a layout number taken from nowhere. | Not in the recorded table. Either match the reference or record the row. |
| 4 | VI-006 — bottom bar replaces the sidebar | "It is a replacement, not a hidden element (annotation B1)." | Both `<aside class="hidden … lg:block">` and `<nav … lg:hidden>` ship in every HTML response (both are present in `.next/server/app/index.html`); CSS hides one. | Informational | Satisfied in behaviour and in the a11y tree (`display:none` removes the hidden one, so only one `aria-label="Primary"` landmark is ever exposed). Noted so the "replacement" wording is not read as a DOM claim. *No action.* |

VI-002, VI-007, VI-008, VI-009, VI-010, VI-011, VI-012 I independently confirmed from source and compiled CSS. VI-013 I confirmed structurally (script present pre-paint, `localStorage` round-trip in `ThemeToggle.tsx:33-42`) but not by reloading a browser.

## Findings

### F1 — An unexpected `/api/health` body crashes the entire shell — BLOCKING

`ApiHealthIndicator.tsx:32` casts the parsed response to `{ api?: ApiHealth }` — an unchecked assertion over arbitrary network JSON — and stores it: `setState(payload.api ?? "unreachable")`. Line 42 then does `const { label, dot, text } = PRESENTATION[state];`. `PRESENTATION` (lines 17-22) has exactly four keys. If `payload.api` is any other string — a captive portal or corporate proxy answering `/api/health` with its own JSON, a future contract value, a typo in a later BFF edit — `PRESENTATION[state]` is `undefined` and destructuring it throws `TypeError` **during render**.

This component is mounted from `Header`, which is mounted from `app/layout.tsx` — the root layout. A throw there is not a degraded badge; it escalates to the App Router's global error boundary (the build emits `.next/server/app/_global-error.html` for exactly this) and the whole application goes to an error page. Spec US1 scenario 2 requires the opposite in so many words: *"the page still renders, and no stack trace or unhandled exception reaches the browser."* The `?? "unreachable"` guard covers a *missing* field and gives the false impression the value is validated; it does nothing about a *wrong* one.
*Action: implementer — narrow the value before using it as a key (validate against the three contract words, or fall back: `PRESENTATION[state] ?? PRESENTATION.unreachable`). Re-request certification for phase 4.*

### F2 — A missing `FITFORGE_API_BASE_URL` is reported as `unreachable` and logged nowhere — BLOCKING

`api-client.ts:25-29` throws a carefully written, actionable error naming the variable and pointing at `.env.example`. `route.ts:20-25` catches it and discards it — no `console.error`, no re-throw, nothing. The comment there argues the browser has no fourth word, which is true and fine for the *response*; it does not follow that the *server* should stay silent.

Verified empirically. I ran the built app with no `FITFORGE_API_BASE_URL` in the environment:

```text
$ npx next start -p 3942
$ curl -s -i http://127.0.0.1:3942/api/health
HTTP/1.1 200 OK
cache-control: no-store
{"api":"unreachable","checkedAt":"2026-09-10T03:47:32.040Z"}
$ tail /tmp/noenv.log       # server output
▲ Next.js 16.3.4
✓ Ready in 516ms
```

Nothing. The operator's only signal is a badge saying the API is unreachable, so they go and check the API process — which is running fine. That is the wrong-process misdirection contract §2 was written to prevent, arriving through a different door, and it directly undermines SC-001 (clone to running by following `docs/onboarding.md` alone) for anyone who mistypes or forgets the variable. As written, the error string in `api-client.ts` is unreachable text: no code path can ever surface it to a human.
*Action: implementer — log the caught error server-side (`console.error`) before mapping to `unreachable`. The response shape stays exactly as it is.*

### F3 — T040 and T041 were not implemented: the BFF route handler has no tests — BLOCKING

`tasks.md` T040 names the file: `src/app/api/health/__tests__/route.test.ts`. It does not exist — `git ls-files` shows only `src/lib/__tests__/health.test.ts` and `src/lib/__tests__/units.test.ts`. T041 ("a test asserting the route never returns a non-200 status of its own") has no implementation anywhere.

What that leaves untested: the always-200 rule, the `cache-control: no-store` header, `export const dynamic = "force-dynamic"`, and the config-error branch of F2. The 18 tests that exist all target `classifyStatus`/`probeApiHealth` — the layer below the thing the browser actually calls. `contracts/health.md` §4 asks for "an automated test per row of the mapping table" from the **consumer**, and the consumer of that contract is `GET /api/health`, not an internal helper.

Nothing about this is hard: the route exports a plain `GET()` that runs in the node environment vitest is already using. I verified every one of those rows by hand instead, running the built app against a stub upstream:

```text
upstream 200          -> {"api":"ready",       …}   HTTP 200
upstream 503          -> {"api":"degraded",    …}   HTTP 200
upstream 404          -> {"api":"unreachable", …}   HTTP 200
upstream 500          -> {"api":"unreachable", …}   HTTP 200
connection refused    -> {"api":"unreachable", …}   HTTP 200
no env var configured -> {"api":"unreachable", …}   HTTP 200
```

The behaviour is correct today. It is simply not defended, and a hand-run I did once is not regression coverage.
*Action: implementer — add `src/app/api/health/__tests__/route.test.ts` per T040 and T041, or amend `tasks.md` with recorded owner approval in a governance commit that precedes the re-commit.*

### F4 — `data-state` is missing on the only data surface in the diff — BLOCKING (or record the exemption)

`docs/rulebooks/frontend-rules.md` § UI States, both items are MUSTs:

> Every data surface MUST implement all states that apply: loading, empty, error/unavailable, and populated — and the error state MUST be visually and programmatically distinguishable from empty.
> Every data surface MUST carry a `data-state` attribute naming the active state (`loading` / `empty` / `error` / `ready`), so tests assert the state rather than guessing from rendered text.

`grep -rn "data-state" src/` → nothing. `ApiHealthIndicator` fetches from `/api/health` and renders four states; it is a data surface by any reading, and the only programmatic handle on its state is the visible label text — exactly what the rule exists to stop. Definition of Done gate 5 makes rulebook compliance part of the review and says any FAIL blocks the phase. Related and smaller: the same file types the response inline as `{ api?: ApiHealth }` without citing `contracts/health.md`, where integration-rules requires "both sides cite the contract file in a comment" and frontend-rules requires response types to "mirror the feature's contract … and cite it".
*Action: implementer — add `data-state={state}` and cite the contract, or the owner records an explicit exemption on the grounds that a header badge is not a "data surface". Note separately that `docs/rulebooks/` contains only `compliance-checklist-template.md` — no instantiated frontend checklist exists, so gate 5's checklist arm currently has nothing to run.*

### F5 — VI-004 hover deviation recorded as PASS — CONFIRM

Row 1 of my deviation table above. `NavLink.tsx:62` adds `hover:text-accent-foreground`; the prototype's inactive item has `hover:bg-accent` and nothing else, and VI-004 says "`--accent` background on hover **only**". `tasks.md` records VI-004 as `PASS — active rgb(244,244,245) on rgb(24,24,27) weight 500; inactive rgb(113,113,122)`, which measures the *resting* colours and never measures the hover state. The verdict is not wrong about what it measured; it is wrong about what VI-004 says.
*Action: owner — decide. Either drop the class (one-token change) or approve the row into the deviation table. Also worth noting the measured `rgb(113,113,122)` is the dark-theme `--muted-foreground` (240 5% 65%), so VI-004 appears to have been graded in dark only.*

### F6 — Six navigation icons invented against a reference that has none — CONFIRM

`navigation.ts:23-39` assigns lucide icons to all six destinations. The prototype's `today` sidebar has no icons. Constitution II: *"When visual references exist, new UI layouts MUST NOT be invented"*, and CLAUDE.md repeats it. The mitigating fact is real — VI-005 (rung 3) says "0.625rem gap between **icon** and label", so the spec plainly expects icons — but that makes this a rung-1/rung-3 conflict, and the conflict rule is *stop and report*, not *pick the lower rung and record a PASS*. The choice of six specific glyphs is in no rung at all.
*Action: owner — ratify the six icons explicitly (they are sensible), or remove them. Either way the resolution belongs in the deviation table, not in a commit message.*

### F7 — Container width and vertical padding taken from nowhere — CONFIRM

`layout.tsx:38` uses `max-w-6xl` (1152px) and `py-4`; the prototype's `<main>` is `max-w-[1400px] px-4 py-6`. The header inner (`Header.tsx:14`) is likewise `max-w-6xl` where the prototype chrome header is `max-w-[1400px]`. No VI item pins either number, so the recorded table had nothing to grade — which is how it slipped through. On a wide monitor the built shell is visibly narrower than the reference.
*Action: owner — match the reference, or record the narrower container as an approved deviation so feature 002 does not have to guess which is authoritative.*

### F8 — The health indicator does not exist below 640px — CONFIRM

`ApiHealthIndicator.tsx:46` — `"hidden items-center gap-2 text-xs sm:inline-flex"`. Below Tailwind's `sm` (40rem = 640px, confirmed in the compiled CSS: `@media (min-width:40rem){.sm\:inline-flex{display:inline-flex}}`) the element is `display:none`, so it is absent from the render *and* from the accessibility tree. T038 says "Wire the health indicator into the shell header, rendering `ready`/`degraded`/`unreachable` distinctly" with no viewport qualifier, US1's acceptance scenarios describe the indicator without one, and frontend-rules § Styling is emphatic: *"Every screen MUST be usable at 375px wide. Why: this is a phone app used standing at a rack; the desktop layout is the secondary case."* On the primary target device the API's status is simply not reported. Note it still costs the request — the effect runs and fetches `/api/health` at every viewport; only the output is hidden.
*Action: owner/implementer — show at least the status dot below `sm` (the label can stay `sr-only`), or record the desktop-only decision.*

### F9 — The timeout test cannot fail — MINOR

`health.test.ts:109-116` ("applies the contract's timeout") asserts two things: that `init?.signal` is an `AbortSignal`, and that the module constant `HEALTH_TIMEOUT_MS === 10_000`. Neither connects the two. Change `health.ts:62` to `AbortSignal.timeout(1000)` and the test still passes; delete `timeoutMs` from the call entirely and it still passes, because `AbortSignal.timeout(undefined)` still returns an `AbortSignal`. The "reports unreachable on a timeout" test fabricates a `DOMException` rather than exercising the mechanism, so nothing in the suite touches the real wiring.

I verified the real behaviour myself against a stub that accepts the connection and never responds:

```text
elapsed_ms=10395
body={"api":"unreachable","checkedAt":"2026-09-10T03:47:57.904Z"}
```

10.4 s — the contract's 10 s plus overhead, correct. But that is my measurement, not the suite's.
*Action: implementer — assert the relationship (e.g. call with `timeoutMs: 50` against a fetch that never settles and assert the promise resolves `unreachable` well inside the default), the same way `tasks.md` T049 argues for the API side.*

### F10 — `health.ts` sits outside the server-only guard, so only the address is protected — MINOR

The split is clean in one direction and leaky in the other. `api-client.ts` carries `import "server-only"` and holds the address; `health.ts` holds `probeApiHealth`, which will call any host it is handed, and carries no guard. `ApiHealthIndicator.tsx:5` imports from it with `import type`, which `isolatedModules` erases — and the bundle grep confirms nothing leaked today. But the guard the plan describes ("a client component importing it fails the build rather than leaking the API's address into the bundle") protects the *address*, not the *capability*. A future `import { probeApiHealth } from "@/lib/health"` in a client component compiles, ships, and lets browser code call an arbitrary API host — which frontend-rules forbids ("Browser code MUST NOT call `fitforge-api` directly") with nothing mechanical standing in the way. The reasoning for the split (pure rules are testable) is sound; the conclusion that the rules must therefore be unguarded is not — the guard could sit on the impure `probeApiHealth` while `classifyStatus` stays free.
*Action: implementer/owner — consider moving `probeApiHealth` behind the guard and leaving `classifyStatus` + types in the pure module, or accept the risk explicitly.*

### F11 — Button hand-rolled against an explicit instruction, and the codebase already wants `asChild` — CONFIRM

Three things point the other way from `button.tsx`:

1. The prototype's own header comment (rung 1/2): *"Components drawn here (button, card, badge, input, tabs, table, progress, sheet) are the ShadCN components of the same name — **do not hand-roll them**."*
2. `frontend-rules.md` § Structure: *"shadcn/ui components are **vendored** into the repository… A vendored component MAY be edited; the edit MUST be noted in the feature's `plan.md`."* This edit is noted in a commit message. `plan.md` was not amended, and plan §4.6 still describes shadcn as the generator that produces `src/components/ui/`.
3. The claim that "nothing needs it yet" is already false. `Header.tsx:22-29` renders a `<Link>` carrying `size-10 … border border-border … hover:bg-accent hover:text-accent-foreground` — the `secondary` + `icon` variant, hand-copied onto an anchor, differing only in `rounded-full`. That is the single case `asChild` exists for, and the file's own escape hatch (`buttonVariants()`) is not used either, so the variant strings are now duplicated in two places and will drift.

The Radix-avoidance argument has merit and `plan.md` §5 genuinely does not approve Radix. But the decision belongs in `plan.md` where later features will read it, not in a commit message nobody greps.
*Action: owner — amend `plan.md` §4.6 to record "Button is hand-written, no Radix, no asChild" as the architecture, and have `Header.tsx` compose `buttonVariants({ variant: "secondary", size: "icon" })` instead of restating it.*

### F12 — `button.tsx` misquotes the annotation it cites — DOC DRIFT

The doc comment (lines 10-14) and the phase 3 commit message both say: *"Annotation H2 on the session screen … reads 'never smaller; this screen is used with sweaty thumbs'. So `h-10` is the floor for anything a member taps mid-set."* The actual annotation, `fitforge-prototype.html:752`, reads:

> H2 **Inputs** are 40px tall **with 16px text** — never smaller; this screen is used with sweaty thumbs.

It is a rule about inputs, and it specifies a text size the Button does not meet — `text-sm` is 14px, not 16px. The height itself is correct and independently required by VI-008, so no code needs to change for buttons; but the citation manufactures authority the source does not grant, and the next person to build the session screen's inputs will read this comment and think H2 is already satisfied.
*Action: implementer — correct the comment to cite VI-008, and leave H2 for the input primitive that actually owes it.*

### F13 — `checkedAt` records when the check started, not when it finished — MINOR

`health.ts:56` stamps `checkedAt` before the fetch. On the timeout path that makes it 10 seconds stale by the time the browser receives it: my measured run stamped `03:47:57.904Z` and delivered the response at roughly `03:48:08`. The contract fixes the *format* (§3: UTC, ISO 8601, `Z`) but not the semantics, and a consumer reading a field named `checkedAt` will reasonably take it as "when this answer was true".
*Action: implementer — stamp after the call returns, or document the choice in the contract. Owner's call which.*

### F14 — The indicator's visual hierarchy is inverted, and `unreachable` is visually identical to `checking` — MINOR

`ApiHealthIndicator.tsx:17-22`: `checking` and `unreachable` share `bg-muted-foreground` + `text-muted-foreground` — the same dot, the same text colour. Only the words differ. Meanwhile `degraded` gets `bg-destructive` + `text-destructive`, the loudest treatment in the set. So the state that means "your API is not answering at all" looks exactly like the state that means "still loading", and the less severe state is the one that shouts. On the colourblind question specifically: the design is *safe*, because every state carries a distinct text label and the dot is decorative (`aria-hidden`) — colour is never the only channel. The problem is severity ranking, not colour discrimination. The deliberate omission of `aria-live` is defensible as argued (it settles once, unprompted announcements interrupt); `title={label}` duplicating already-visible text adds nothing but is harmless. Unmount safety is handled correctly — `controller.abort()` in cleanup, and the `.catch` re-checks `controller.signal.aborted`, which also makes it StrictMode-double-invoke safe.
*Action: implementer — give `unreachable` its own treatment distinct from `checking`, and consider whether `degraded` really outranks it.*

### F15 — No `color-scheme` declaration — MINOR

`grep color-scheme` over the compiled CSS returns nothing, and `globals.css` declares none. In dark theme the browser renders native scrollbars, focus rings on native controls, date pickers and `<select>` popups in light. The prototype omits it too, so this is not a deviation from the reference — but the reference is a static page with one checkbox, and feature 002 ships forms.
*Action: implementer, at feature 002 — add `color-scheme: light` on `:root` and `dark` on `.dark`. Not a phase 3/4 blocker.*

### F16 — The Visual Compliance Loop record has no screenshots, so its result is not auditable — CONFIRM

`docs/sdlc/review-process.md` step 1 says capture the render, and the exit rule says *"Attach the final table and both screenshots to the phase notes — the AI review verifies they exist."* They do not exist; `screenshots/` holds only the prototype. Reading computed styles instead of eyeballing is genuinely better evidence for the numeric items, and I do not want to discourage it — but it is not a substitute, and it is exactly why F5 and F7 got through: computed-style reads answer the questions you thought to ask, and a side-by-side capture surfaces the ones you did not. I am recording plainly, as instructed, that **I could not independently confirm the recorded PASS verdicts**; I confirmed the statically checkable subset and found three problems in the rest.
*Action: implementer — attach the two captures (≥1024px and 375px, both themes) to the phase notes before merge, so gate 6's human reviewer can do what I could not.*

### F17 — The phase 4 commit message overstates its test count — MINOR

"Twenty tests, one per row of the mapping table plus…". `vitest run` reports 20 total across 2 files, but 2 of those are the pre-existing `units.test.ts` cases that predate the branch. Phase 4 added 18.
*Action: none — noted so the number is not carried into the phase record as if 20 were new.*

### F18 — `/api/health` is unauthenticated, uncached, and fans out one 10 s upstream call per request — MINOR

Every hit on `/api/health` issues a fresh upstream request with `cache: "no-store"`, `dynamic = "force-dynamic"`, no rate limit, no in-flight de-duplication and no short cache window. When the API is hanging, each of those occupies a connection for 10.4 s (measured). Nothing secret is disclosed — the response is three words, correctly — and the contract explicitly accepts that a readiness probe is broadly reachable. But the BFF is a public origin and the API is not, so this route is an amplification hop that did not exist before.
*Action: owner — decide whether a 1-2 s in-flight cache belongs here now or with the first authenticated feature. No change required for this phase.*

### F19 — No AI code review exists on file for phases 1, 2 or 6 — MINOR (process)

`specs/001-solution-scaffold/` contains `spec.md`, `plan.md`, `tasks.md`, `contracts/`, `screenshots/` and `human-pr-review.md` — no `ai-code-review*.md` of any kind before this file. Definition of Done gate 5 applies to **every** phase commit, and four certified or pending phases in `fitforge-api` (1, 2, 6) plus phase 5 in governance have none. `human-pr-review.md` itself flags this ("**AI review**: … NOT YET RUN"). This review covers `fitforge-web` phases 3 and 4 only and must not be read as covering the rest.
*Action: owner — commission the `fitforge-api` review before merge; gate 6 depends on gate 5 having been satisfied per phase.*

## Constitution re-check (post-implementation)

**FAIL** — one principle is engaged and not satisfied; the rest hold.

- **I. Specification First** — PASS. `spec.md`, `plan.md`, `tasks.md` and the contract all predate both phase commits (`5a34b5d` precedes `e0df410`).
- **II. Source of Truth Hierarchy** — **FAIL.** Three divergences from rung 1/2 (F5 hover colour, F6 invented icons, F7 container width) were resolved in favour of the implementation without being reported as conflicts, and one of them is recorded as a PASS. The plan-time check said "no conflict. The prototype is rung 2 and this plan transcribes it rather than reinterpreting it" — that held for the tokens, which are exact, and did not hold for the markup. The conflict rule is *stop and report*; F6 in particular is a genuine rung-1-versus-rung-3 conflict (the prototype shows no icons, VI-005 presumes one) that should have surfaced as a question, not a choice.
- **III. Repository Separation** — PASS. Both phases touch only `fitforge-web`. Phase 4's end-to-end run found a defect in `fitforge-api` and correctly refused to fix it across the boundary, queueing phase 6 instead — that is the principle working as intended, and it is the best judgement call in the diff.
- **IV. Architecture Consistency** — PASS with a caveat. No package outside plan §5; ADR §4.6's server-component default is honoured throughout. The caveat is F11: a real architectural decision (no Radix, no `asChild`, hand-written primitives) was made and recorded only in a commit message, where constitution IV expects `plan.md`.
- **V. Domain Invariants** — N/A structurally, PASS in substance. No domain rule ships. `units.ts` converts for display only and cites invariant §4; nothing in the BFF computes or decides anything, satisfying integration-rules' "the BFF is not a tier that owns anything".
- **VI. Security** — PASS. The plan-time claim ("`FITFORGE_API_BASE_URL` … names in `.env.example` and values in the environment") is verified true in the built output, not merely asserted. F18 is a load consideration, not a disclosure.
- **VII. External Integration Governance** — PASS on the contract, partial on verification. The contract predates both sides and both server-side modules cite it by section. But §4's consumer verification ("an automated test per row … including the timeout row") is satisfied one layer below the consumer, and the route itself has none (F3), while the timeout row's test cannot fail (F9).
- **VIII. Testing Requirements** — engaged late. The plan itself named the two things owing coverage: "the readiness check's failure path and **every row of the BFF's reachability mapping table**". The rows are covered as pure functions; the route that serves them is not.
- **IX. Human Review** — pending, correctly. `human-pr-review.md` is scaffolded and entirely unticked.
- **X. Controlled Delivery** — PASS. One phase per commit, `phase N` token present in both subjects, scope check green at exit 0, phase 4 independently revertible, and `ci-held` correctly declared before the first phase with the agent explicitly disclaiming certification in both commit messages.

## Test coverage observed

`npm test` → `vitest run` v5.0.0, **2 files, 20 tests, all passing, 479ms**. No `vitest.config.*` exists, so the default node environment and default include glob apply — which is why there is no component test anywhere and no path to one without adding jsdom.

- **`src/lib/__tests__/health.test.ts` — 18 tests, added by phase 4.**
  - `classifyStatus` (8): 200→`ready`; 503→`degraded`; an `it.each` over `[500, 502, 404, 401, 301]`→`unreachable` (5 cases); and `expect(classifyStatus(503)).not.toBe(classifyStatus(500))`, which is the one assertion that directly defends the contract's central prohibition against collapsing `degraded` into `unreachable`. That test is well chosen — it fails for the right reason and its comment explains why.
  - `probeApiHealth` (10): the three happy rows; three transport failures (a `TimeoutError` `DOMException`, an `ECONNREFUSED` `TypeError`, an `ENOTFOUND` `TypeError`); "never throws, whatever the transport does"; the exact `checkedAt` string `2026-09-10T01:23:45.000Z` with a `Z`-suffix assertion; `toHaveBeenCalledTimes(1)` for the no-retry rule; and a URL assertion `expect(fetchImpl.mock.calls[0][0]).toBe("http://api.test/health/ready")` from a base with a trailing slash, which covers both "readiness not liveness" and the double-slash case in one. The `vi.fn<typeof fetch>` typing is a deliberate, correct choice and the comment says why.
  - The weak one is the timeout test (F9): it asserts the shape of the argument and the value of an unrelated constant, and no mutation of the timeout wiring would fail it.
- **`src/lib/__tests__/units.test.ts` — 2 tests, pre-existing.** A `toBeCloseTo(220.462, 3)` conversion and a 10-decimal round-trip citing invariant §4. Fine, and unaffected by this branch.
- **Not covered at all**: `src/app/api/health/route.ts` (T040, T041 — F3), `ApiHealthIndicator` (fetch, error path, the four-state map — F1 would have been caught by a two-line test of `PRESENTATION` key coverage), `ThemeToggle` (`useSyncExternalStore` subscription, `localStorage` write, the `catch`), `theme-script.ts` (stored-beats-OS, garbage-value handling), `NavLink`'s active-state predicate (`href === "/" ? pathname === "/" : pathname.startsWith(href)` — note this will mark `/programs` active for a future `/programs-archive` route; no test, and no route yet to expose it), `navigation.ts`'s fixed order (VI-003 is "never reordered" and nothing asserts it — a five-line snapshot would make that inventory item machine-enforced).
- **Manual verification I performed**, since the automated coverage does not reach it: all six route-level mapping outcomes against a stub upstream; the 10.4 s real timeout; the missing-env path; `cache-control: no-store` and HTTP 200 on every path; and the `.next/static` bundle greps.

## Residual risk

The risk concentrates in the two client components, which are the only code here with no test of any kind and the only code that runs where a failure is visible to a member.

**F1 is the one that can hurt in production**: a value the component does not control is used as an object key whose miss throws inside the root layout, so an unexpected `/api/health` body does not degrade the badge — it replaces the application with an error page, contradicting the one acceptance scenario (US1 #2) that this feature exists to prove. It is a one-line fix and there is no reason to carry it.

**F2 carries the operational risk**: the diagnostic the implementer wrote is unreachable, so the first developer to follow `docs/onboarding.md` and mistype an environment variable will be told the API is unreachable while the API is fine. That is a direct hit on SC-001, and SC-001 is measured against a document that has already been written and tested — this defect is the kind that survives precisely because the person who wrote the message never saw it fail to appear.

**F3 and F9 are the coverage risk**: today's behaviour is correct — I confirmed all six route outcomes and the real 10 s timeout by hand — but nothing in the suite would notice if any of it regressed, and a hand-run recorded in a review is not regression coverage. F3 is also a plain, unamended departure from approved tasks.

**F5-F7 and F16 are the visual risk**: the recorded Visual Compliance Loop result is not a fact I could confirm. Its numeric verdicts I re-derived and they hold — the tokens are genuinely exact, and that is the hardest and most valuable part to get right. Its markup verdicts do not: one is wrong (VI-004), and two divergences never entered the table because no VI item happened to pin the number. SC-003 requires an empty or user-approved deviation table at merge, and on my reading the table is not currently empty.

**Before merge**: F1, F2, F3 fixed and re-certified; F4 fixed or exempted on the record; F5, F6, F7 resolved into the deviation table with the owner's decision on each; the two screenshots attached (F16). **After merge is acceptable for**: F9-F15, F17, F18. **Independent of this repository**: F19 — `fitforge-api` phases 1, 2 and 6 still owe a gate-5 review, and gate 6 cannot honestly complete until they have one.
