---
id: 05-18
stage: 05
title: Layout skeleton with fixed heights and internal scrolling
points: 1
owner: web
status: todo
depends_on: [05-17]
files:
  - web/src/App.tsx
  - web/src/App.test.tsx
  - web/src/index.css
---
# 05-18 · Layout skeleton with fixed heights and internal scrolling
**Points:** 1 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 05-17 · **Unblocks:** 05-19, 05-20
## Goal
Render the final layout structure — header, filter placeholder, 340 px left panel plus fluid results, 45vh map
section — inside a shell that never scrolls the whole page.
## Why this task exists
§5.1 of the architecture doc fixes the layout: 56 px header, 64 px filter bar, a 340 px panel, and a full-width
45vh map. A page that scrolls as a whole puts half the map off-screen and makes list and panel fight over one
scrollbar. Retrofitting it after Leaflet (stage 07) is painful, so it is set now. The panes are placeholders —
stages 06–07 fill them — but landmarks, heights and scroll model are final.
## Current state (preconditions)
- 05-17 done: `App.tsx` renders the header totals, the results rows, the `No location (N)` group, the placeholder
  panel and the loading/error copy from `composeView`; `App.test.tsx` has its five cases and is green.
- 05-01 done: Tailwind 3.4 is configured with the tokens from `plan/02-architecture.md` §5.3, and
  `web/src/index.css` holds the three Tailwind directives. No `web/src/components/` yet; 05-19/05-20 slot their
  components in here.
## Read before you start
- `CLAUDE.md` §3 (TDD; class assertions are legitimate — jsdom has no layout), §5 (YAGNI).
- `plan/02-architecture.md` §5.1 — the wireframe and its measured decisions; §5.3 — the style tokens.
- `plan/stage-05-web-shell.md` Task 5.5 — the required structure and its four layout requirements,
  `plan/90-test-matrix.md` row W-17.
## Files to touch
| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/App.tsx` | the shell, landmarks, two placeholder panes, map placeholder |
| modify | `web/src/App.test.tsx` | the five layout cases below |
| modify | `web/src/index.css` | root height and page-scroll base rules only |

Only these three files change; report anything else you believe is needed.
## Test first (red)
The 05-17 cases stay untouched and green; add these next to them.

1. `App.test.tsx::test_semantic_landmarks_exist` — after load there is one `header`, one `main`, one `aside` and
   one `section` whose accessible name is `Results` (by role and `aria-label`), built from semantic elements, not
   `div`s with roles.
2. `App.test.tsx::test_shell_is_full_height_flex_column_without_horizontal_scroll` — the outermost element App
   renders (`container.firstElementChild`) carries `h-screen`, `flex`, `flex-col` and `overflow-x-hidden`, and no
   `min-h-screen`, which would let the page grow and scroll as a whole.
3. `App.test.tsx::test_header_and_filter_placeholder_have_fixed_heights` — the `header` has `h-14` (56 px) and the
   filter placeholder below it has `h-16` (64 px); the placeholder's decorative boxes are `aria-hidden`.
4. `App.test.tsx::test_main_is_a_340px_plus_fluid_grid_with_independent_scroll` — `main` has
   `grid-cols-[340px_1fr]` and `min-h-0`; `aside` and the results `section` each carry `overflow-y-auto` and
   `min-h-0`.
5. `App.test.tsx::test_map_section_is_45vh_with_a_stage_07_placeholder` — the last `section` beneath `main` is
   `h-[45vh]`, full width, containing `map loads in stage 07`.

Expected red: `expect(element).toHaveClass("h-screen")` failing, or
`Unable to find a section with the accessible name: Results`. Run `pnpm -C web test --run App` and copy one
failure line.
## Implement (green)
Restructure the JSX of `web/src/App.tsx`; keep every string, hook and composition step from 05-17 as it is.

- Root element: `h-screen flex flex-col overflow-x-hidden` plus the §5.3 background and text tokens. Nothing
  inside may exceed the viewport width.
- `header`: `h-14 shrink-0`, bottom border, totals line; leave a `StatusBar` slot (05-19), no second freshness
  line.
- Filter placeholder: `h-16 shrink-0`, a row of inert boxes marked `aria-hidden="true"`, no inputs and no labels
  (06-01 owns the real controls).
- `main`: `flex-1 min-h-0 grid grid-cols-[340px_1fr] gap-4 overflow-hidden`; `aside` is the left placeholder
  (`overflow-y-auto min-h-0`, selected name or `No building selected`); the results `section` is
  `aria-label="Results"` with `overflow-y-auto min-h-0`, rendering `inView` rows and the `No location (N)` group.
- Map placeholder: a `section` after `main`, `h-[45vh] shrink-0`, grey block, text `map loads in stage 07`.
- `web/src/index.css`: only base rules — `html, body, #root` full height, `body` `overflow: hidden`, the 14 px
  system-UI base type and `tabular-nums` for prices. No component classes.

Run: `pnpm -C web test --run App` — the layout cases green and the 05-17 cases still green.
## Refactor / keep it DRY
The shell exists once, in `App.tsx`. Do not extract a `Layout` component with one caller (YAGNI), and do not
duplicate the panel containers when 05-19/05-20 insert their components — those go inside the existing
landmarks.
## Verification (how to test it)
```text
pnpm -C web test --run App
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: 10 passed in `App.test.tsx` (5 old, 5 new), suite green, typecheck clean, `make check` green.
## Out of scope (do not do)
- Real filter controls (06-01) and results rows (06-06/06-07).
- Leaflet, tiles or marker layers (07-01/07-02) — the block stays grey text.
- `StatusBar` and `EmptyState` components: 05-19 and 05-20 own them.
- Mobile sheet/toggle (07-12), animations, icon libraries, gradients, dark mode, or changing 05-17's copy.
## Definition of done
- [ ] All five cases exist, named as written, and pass; the 05-17 cases are unchanged.
- [ ] Red run observed and quoted; target tests + lane suite + typecheck + `make check` green.
- [ ] Landmarks are semantic; the shell uses `h-screen` (not `min-h-screen`) and both scroll panes carry `min-h-0`.
- [ ] Only the three listed files plus this task file are modified; `status: done`, `## Result` filled; one trunk
      commit.
## Commit
Message (first line exactly, then body):

```text
05-18: add layout skeleton with internal scrolling

- points: 1
- red: test_shell_is_full_height_flex_column_without_horizontal_scroll failed (expected element to have class "h-screen")
- wrap the app in h-screen flex-col, header h-14, filter placeholder h-16
- main grid-cols-[340px_1fr] with min-h-0 and two independently scrolling panes
- add the 45vh map placeholder, landmarks and base no-page-scroll rules in index.css
- suite: <N> passed
```

Commit only the listed files plus this task file.
## Traps and review notes
- `min-h-screen` (or no height) instead of `h-screen`: the page grows and the document scrolls, defeating the
  layout.
- Missing `min-h-0` on `main` or on a pane: a flex/grid child will not shrink, `overflow-y-auto` never engages,
  and the list pushes the map off-screen.
- Reviewers: grep `web/src/App.tsx` for `min-h-screen` (absent), count `overflow-y-auto` (exactly two panes),
  check `aria-label="Results"`.
## Plan samples are untrusted
The wireframe in `plan/02-architecture.md` §5.1 is ASCII, not markup, and Task 5.5 of the stage plan sketches
class names that predate the scaffold. Apply `CLAUDE.md` §6: read the installed Tailwind version and the tokens
configured in 05-01, then follow the repo. Record any deviation (for example a token class that does not exist)
under `## Decisions`. Add no dependency.
## Decisions

## Result
| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
