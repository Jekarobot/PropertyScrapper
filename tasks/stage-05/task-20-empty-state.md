---
id: 05-20
stage: 05
title: Reusable empty state component with an optional action
points: 0.5
owner: web
status: todo
depends_on: [05-18]
files:
  - web/src/components/EmptyState.tsx
  - web/src/components/EmptyState.test.tsx
  - web/src/App.tsx
  - web/src/App.test.tsx
---
# 05-20 · Reusable empty state component with an optional action
**Points:** 0.5 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 05-18 · **Unblocks:** 06-09, 06-12
## Goal
Add `EmptyState` — title, hint, optional action button — and use it for the loader error `App.tsx` renders
inline today.
## Why this task exists
Three places need the same block: a failed dataset load (this stage), filters with no matches (06-12) and an
empty viewport (07-10). One component keeps copy, heading level and button semantics identical. `App.tsx`
renders that error inline from 05-17, so swapping the markup without changing the user-visible contract proves
it fits.
## Current state (preconditions)
- 05-18 done: the layout skeleton renders header, `main`, `aside`, the results section and the map placeholder;
  `App.test.tsx` has ten green cases, including 05-17's error case.
- `App.tsx` renders that error inline: title `Could not load the dataset`, hint
  `Check that the dataset was published, then try again.`, `Try again` button calling `reload`;
  `web/src/components/` exists if 05-19 is done.
## Read before you start
- `CLAUDE.md` §3 (TDD), §5 (YAGNI — no variant system, no render props).
- `plan/stage-05-web-shell.md` Task 5.5 — props `{ title, hint, action? }` and the requirement that stages 06/07
  reuse it.
- `plan/02-architecture.md` §5.1 — the no-buildings copy and reset button as the future caller's shape.
## Files to touch
| Action | Path | What goes there |
|---|---|---|
| create | `web/src/components/EmptyState.tsx` | the component and its props type |
| create | `web/src/components/EmptyState.test.tsx` | the three cases below |
| modify | `web/src/App.tsx` | replace the inline error markup with the component |
| modify | `web/src/App.test.tsx` | the integration case below |
Only these four files change; report anything else you believe is needed.
## Test first (red)
1. `EmptyState.test.tsx::test_renders_title_and_hint` — `title: "No buildings in this area"`,
   `hint: "Zoom out or clear the filters."` → title queryable as a heading with that name, hint visible.
2. `EmptyState.test.tsx::test_renders_an_action_button_that_calls_the_callback` —
   `action: { label: "Clear filters", onClick }` → a `button` named `Clear filters`; a `userEvent` click calls
   `onClick` exactly once.
3. `EmptyState.test.tsx::test_renders_no_button_without_an_action` — no `action` prop → no button in the tree
   (`queryByRole("button")` null) while title and hint still render.
4. `App.test.tsx::test_loader_error_uses_empty_state_with_retry` — HTTP 500 → heading `Could not load the dataset`,
   hint `Check that the dataset was published, then try again.`, button `Try again`; clicking issues a second
   request and, on success, the rows appear. Same contract as 05-17. Expected red:
   `Failed to resolve import "./EmptyState" from "src/components/EmptyState.test.tsx"` for cases 1–3, and the
   missing-heading assertion for case 4. Run `pnpm -C web test --run EmptyState` then `--run App`; copy one failure
   line.
## Implement (green)
Create `web/src/components/EmptyState.tsx`:
- Props: `title: string`, `hint: string`, `action?: { label: string; onClick: () => void }`. No children, icon,
  variant or `className` passthrough.
- Markup: title as an `h2`, hint as a paragraph, and — only with `action` — a real `<button type="button">` named
  `action.label` wired to `action.onClick`; no click handler on a `div`.
- Styling: centered, muted text, §5.3 tokens only. One-sentence docstring: it renders a title, a hint and an
  optional action for empty or failed states. Then replace the inline loader-error markup in `App.tsx` with it,
  passing the three strings from 05-17 and `reload`; leave the loading placeholder untouched. Run:
  `pnpm -C web test --run EmptyState` and `pnpm -C web test --run App` — all green.
## Refactor / keep it DRY
One empty-state markup block, in this component; the error copy lives once, in the `App.tsx` call site. A second
caller passes its own `title`/`hint`/`action` — never a `variant`.
## Verification (how to test it)
```text
pnpm -C web test --run EmptyState
pnpm -C web test --run
pnpm -C web typecheck
make check
```
Expected: 4 passed, suite green, `tsc --noEmit` clean, `make check` green.
## Out of scope (do not do)
- The filter no-match copy (06-12) and the map empty state (07-10): they consume this later.
- Changing the strings, the hook or any 05-17/05-18 `App.test.tsx` case; a new dependency of any kind.
## Definition of done
- [ ] All four cases exist, named as written, and pass.
- [ ] Red run observed and quoted; target tests + lane suite + typecheck + `make check` green.
- [ ] No `any`; the error copy is not duplicated in the component; only the four listed files plus this task file are
      modified.
- [ ] `status: done`, `## Result` filled; exactly one trunk commit with the message below.
## Commit
Message (first line exactly, then body):
```text
05-20: add reusable empty state component

- points: 0.5
- red: test_renders_title_and_hint failed (Failed to resolve import "./EmptyState")
- add EmptyState with title, hint and optional action button
- use it for the App loader error; the 05-17 accessible contract is unchanged
- suite: <N> passed
```
Commit only the listed files plus this task file.
## Traps and review notes
- Typing `action` as a `ReactNode` makes the accessible name untestable and lets a caller pass a link:
  `{ label, onClick }` is the contract; a clickable `div` instead of `<button type="button">` loses keyboard
  access.
- Reviewers: grep `EmptyState.tsx` for `Could not load` (absent), for `onClick` on non-button elements (absent),
  and confirm one `<button` guarded by `action`.
## Plan samples are untrusted
Task 5.5 names the props but shows no markup; apply `CLAUDE.md` §6 to the §5.3 tokens and the installed
`@testing-library/user-event` API, and record deviations under `## Decisions`. Add no dependency.
## Decisions
## Result
| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
