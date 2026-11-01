---
id: 05-15
stage: 05
title: Viewport bounding-box hook with whole-city default
points: 0.5
owner: web
status: todo
depends_on: [05-11]
files:
  - web/src/state/useViewport.ts
  - web/src/state/useViewport.test.ts
---
# 05-15 · Viewport bounding-box hook with whole-city default
**Points:** 0.5 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 05-11 · **Unblocks:** 05-17, 07-04
## Goal
Add `useViewport`, the one piece of state holding the current map bounding box, where `null` means "whole city".
## Why this task exists
The right-hand list shows only buildings inside the current map view (rule F6, `plan/02-architecture.md` §5.2),
so the app must hold "what the map currently shows" before any map exists. `null` is the honest default: until
the map reports a box, nothing is cut and the whole city is listed — no magic coordinates, no second definition
of "everywhere". Stage 07's map will call the setter on `moveend` (07-04); a stable setter identity is what
stops that subscription from being torn down and recreated on every render.
## Current state (preconditions)
- 05-11 done: `web/src/lib/viewport.ts` exports the `BBox` interface with `south`, `west`, `north`, `east` (all
  `number`), plus `inBBox` and `splitByLocation` with its own tests. Import that type; do not define a second box
  shape.
- 05-01/05-02 done: Vitest + jsdom + `@testing-library/react` wired, `pnpm -C web test --run` green.
  `web/src/state/` exists from 05-13; no `useViewport.ts` yet.
## Read before you start
- `CLAUDE.md` §3 (TDD), §5 (no `any` in `web/src/state/**`; one implementation per rule).
- `plan/stage-05-web-shell.md` Task 5.4 — `bbox: BBox | null` and the comment "null = whole city"; Task 5.3 —
  viewport logic is Thailand-only, no dateline handling.
- `plan/02-architecture.md` §5.2 rules F6 and F8 — what the box is used for and when it must update.
- `web/src/lib/viewport.ts` — the `BBox` field order and the `splitByLocation` signature this hook feeds.
## Files to touch
| Action | Path | What goes there |
|---|---|---|
| create | `web/src/state/useViewport.ts` | the hook |
| create | `web/src/state/useViewport.test.ts` | the four cases below |

Only these two files change; report anything else you believe is needed.
## Test first (red)
Use `renderHook` from the installed `@testing-library/react`; nothing renders the app.

1. `useViewport.test.ts::test_initial_bbox_is_null_meaning_whole_city` — first render → `bbox` is `null` and
   `setBBox` is a function.
2. `useViewport.test.ts::test_set_bbox_stores_the_box` —
   `setBBox({ south: 13.7, west: 100.5, north: 13.75, east: 100.55 })` → `bbox` deep-equals that object; the box is
   the one that contains the documented coordinates 13.7187616 / 100.5202356 (`plan/02-architecture.md` §3.3), so
   the four numbers are test input, not new product data.
3. `useViewport.test.ts::test_set_bbox_null_clears_back_to_whole_city` — set a box, then `setBBox(null)` → `bbox`
   is `null` again; a second `null` call keeps it `null`.
4. `useViewport.test.ts::test_set_bbox_identity_is_stable_across_renders` — capture the function reference on the
   first render, set a box, re-render → the reference is unchanged (`toBe`). This is what lets 07-04 depend on the
   setter without re-subscribing.

Expected red: `Failed to resolve import "./useViewport" from "src/state/useViewport.test.ts"` for every case;
run `pnpm -C web test --run useViewport` and copy one failure line.
## Implement (green)
Create `web/src/state/useViewport.ts` with one hook:

- `useViewport()` → `{ bbox, setBBox }`; `bbox` is `BBox | null`, `setBBox(b: BBox | null): void`. `BBox` comes
  from `lib/viewport.ts`.
- Initial state is `null` — "whole city" — and clearing means setting `null`, not an empty or zero-sized box.
- The hook stores the box unchanged: no clamping, no validation of `south < north`, no padding, no conversion to
  Leaflet bounds.
- `setBBox` keeps a stable identity across renders. One-sentence docstring: it holds the current map box, `null`
  means no cut.

Run: `pnpm -C web test --run useViewport` — four green.
## Refactor / keep it DRY
`BBox` has exactly one definition (`lib/viewport.ts`) and exactly one consumer of "no cut" semantics: `App.tsx`
computes `bbox ?? whole-city box` in 05-17. Do not add a `WORLD` constant, a default box, or coordinate maths
here — that would be a second opinion about "everywhere".
## Verification (how to test it)
```text
pnpm -C web test --run useViewport
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: 4 passed, suite green, `tsc --noEmit` clean, `make check` green.
## Out of scope (do not do)
- Leaflet, map components, `moveend` subscriptions, throttling — 07-01…07-05.
- A `WORLD`/default box here, coordinate validation, clamping or a "bounds to bbox" converter.
- URL sync, `localStorage`, or storing the map object itself (F10).
## Definition of done
- [ ] All four cases exist, named as written, and pass.
- [ ] Red run observed and quoted in the commit body; target tests + lane suite + typecheck green.
- [ ] No `any`; `BBox` imported, not restated; only the two listed files plus this task file are modified.
- [ ] `status: done`, `## Result` filled; exactly one trunk commit with the message below.
## Commit
Message (first line exactly, then body):

```text
05-15: add viewport bbox hook

- points: 0.5
- red: test_initial_bbox_is_null_meaning_whole_city failed (Failed to resolve import "./useViewport")
- add useViewport holding BBox | null, null meaning whole city
- stable setBBox identity for the stage 07 moveend subscription
- suite: <N> passed
```

Commit only the listed files plus this task file.
## Traps and review notes
- Defaulting to a world-sized box here instead of `null` puts a second definition of "everywhere" in the app and
  makes 05-17's `bbox ?? WORLD` meaningless.
- Creating the setter inline (`onBBox={b => setBBox(b)}` style) breaks case 4; wrap it once.
- Validating or clamping the box hides map bugs that should be visible; the box is stored as given.
- Reviewers: grep `useViewport.ts` for `leaflet`, `Math.`, and any four-number literal — none may appear.
## Plan samples are untrusted
`plan/stage-05-web-shell.md` Task 5.4 shows the signature only. Apply `CLAUDE.md` §6: read the real `BBox`
interface from 05-11 and the installed `@testing-library/react` types for `renderHook`/`act`. If the repo
disagrees with the sample, follow the repo and record it under `## Decisions`. Add no dependency.
## Decisions

## Result
| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
