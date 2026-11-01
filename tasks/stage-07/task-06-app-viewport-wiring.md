---
id: 07-06
stage: 07
title: Viewport drives the results list
points: 1
owner: web
status: todo
depends_on: [07-05, 05-17]
files:
  - web/src/App.tsx
  - web/src/App.test.tsx
---

# 07-06 · Viewport drives the results list

**Points:** 1 · **Stage:** 07 — Map and selection · **Owner lane:** web · **Depends on:** 07-05, 05-17 · **Unblocks:** 07-09

## Goal

Mount `MapView` in the app and connect its coalesced `moveend` to `setBBox`, so dragging the map cuts the right-hand list to the buildings inside the new view — one filtered list in `App`, one source of truth, no second filter in the map.

## Why this task exists

Every piece exists but nothing joins them: `useViewport` holds a `null` bbox nobody sets, `composeView` already cuts by `bbox ?? WORLD`, and `MapView` reports bounds to a callback the app does not pass. Without this wiring the map is a picture and the right-hand list ignores it — the product's defining interaction ("move the map, the list follows") is missing (F5, F8). It also fixes the ownership rule that keeps the rest of stage 07 honest: the map receives the **filtered** buildings (so a bubble exists for every filtered building, whether or not it is on screen), the list receives the **viewport-cut** ones, and neither recomputes the other.

## Current state (preconditions)

- 07-05 done: `MapEvents` coalesces `moveend` and calls `onViewportChange` with a `BBox`; `MapView` renders it.
- 05-17 done: `App.tsx` owns `composeView(dataset, filters, bbox)` returning `{ units, buildings, inView, noLocation }`, with `bbox` from `useViewport` and `null` meaning whole city; `App.test.tsx` has the 05-17 cases and stubs `fetch` with the sample fixture.
- 05-15 done: `useViewport` returns `setBBox` with a stable identity.
- 05-18/05-19/05-20 done by lane order: `App` has the layout with the bottom map band (45 vh) and the results region; if the band does not exist, stop and report — do not invent a second layout.
- `App.tsx` does not import `MapView` yet, and `setBBox` is never called.

## Read before you start

- `CLAUDE.md` §3, §5 (single source of truth: `App` composes state once; components are dumb), §6.
- `plan/02-architecture.md` §5.1 — the map is the bottom band, the right list is the viewport-filtered buildings; §5.2 rows F5 (filters before the viewport) and F8 (200 ms).
- `plan/stage-07-map.md` Task 7.2 items 5–6 — the App integration and the inclusive edge case.
- `plan/90-test-matrix.md` rows W-05 (filter-before-viewport, from 05-17), W-08 (this task's list half) and W-06.
- `web/src/App.tsx` and `web/src/App.test.tsx` as they stand — the 05-17 `composeView` and the `WORLD` constant.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/App.tsx` | mount `MapView`, pass `onViewportChange={setBBox}`, keep one composition |
| modify | `web/src/App.test.tsx` | the six cases below, with `MapView` mocked |

Nothing else may change: no `MapView`, no `MapEvents`, no hook, no `lib/` edit, no panel.

## Test first (red)

Mock `MapView` with `vi.mock` so a case can read the props the app passed and invoke them inside `act`. The fixture's coordinates are the only coordinates used.

1. `test_bbox_around_bangkok_shows_only_buildings_in_view` — render `<App />`, invoke `onViewportChange` with a box containing only `p00348` (use the fixture's own point ± a small delta): the results list shows one row, `Premier Thonglor` is gone, and the header total (`6 units`) still reports the dataset.
2. `test_map_receives_the_same_filtered_buildings` — the `buildings` prop passed to `MapView` is the filtered/grouped list from `composeView` (three buildings, including the null-coordinate one), while the list receives only `inView`; the map is never handed `inView` and never filters by coordinates itself.
3. `test_building_exactly_on_the_viewport_edge_is_listed` — a box whose north edge equals `p00348`'s latitude (read it from the fixture) includes `p00348`; a box whose north edge is one ten-thousandth lower excludes it. Inclusive on the edge, W-06/W-08's boundary.
4. `test_move_updates_the_list_in_the_same_render_pass` — after a single `act` around `onViewportChange`, the rows already reflect the new box; there is no second effect/paint where the list is briefly stale (assert the DOM right after the act, without an extra `waitFor`).
5. `test_bbox_null_keeps_every_mapped_building` — the initial state calls `MapView` with no viewport cut: both mapped buildings in the list, the unmapped one in the `No location` group; no `setBBox` happened at mount.
6. `test_filters_still_apply_before_the_viewport` — filter to `district: "Khlong Toei"` (only the unmapped building) and then set a wide box: the list stays empty because the cut never re-admits a building the filters removed.

Expected red: `Unable to find an element with the text: The Bangkok Sathorn` after the box is set (the list ignores the bbox), and `MapView` prop `onViewportChange` undefined. Run `pnpm -C web test --run App`, quote one line.

## Implement (green)

- Render `MapView` inside the map region of the layout with exactly four props: `buildings={buildings}` (the filtered, grouped list — **not** `inView`), `selectedId`, `onSelect={select}`, `onViewportChange={setBBox}`. Pass `initialCenter` only if the layout needs a non-default center; the default lives in `MapView`.
- `onViewportChange` is the hook's setter passed straight through — no wrapper that clamps, pads or re-rounds the box, and no local bbox copy in `App`.
- Do not add a second cut, a `useMemo` that re-filters by coordinates, or a `filter` inside the JSX: `composeView` already produced `inView`/`noLocation`, and it stays the only place that cuts.
- Keep `WORLD` as the only "no cut" box.
- If a drag would re-render the list synchronously more often than the coalescing window allows, the fix belongs in `MapEvents` (07-05), not here.

Run: `pnpm -C web test --run App` — six new green; the 05-17, 06-09 and 06-12 cases still green.

## Refactor / keep it DRY

One composition (`composeView`), one bbox source (`useViewport`), one consumer of `setBBox` (the map). `MapView` must not import `splitByLocation`; `App` must not import Leaflet. The map's `buildings` prop and the list's `buildings` prop come from the same memoised composition — two different arrays, one computation.

## Verification (how to test it)

```text
pnpm -C web test --run App
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Manual check: `pnpm -C web dev` → the map sits in the bottom band; dragging it changes the right-hand list once per gesture; zooming in narrows the list; no console errors and no visible flicker.

## Out of scope (do not do)

- The building panel, the selected-marker style, the no-location bucket markup — 07-07…07-10.
- Auto-fit, `fitBounds`, "search this area" buttons, URL sync of the bbox (F10 forbids it).
- Moving the viewport cut into `MapView` or the list.
- A server, API or persisted bbox: the dataset is one static file and the viewport lives in React state (F10).
- Changing `useViewport`, `composeView`'s order, or the 05-17 error/loading states.

## Definition of done

- [ ] All six cases exist, named as written, and pass; the 05-17/06-09/06-12 `App` cases still pass unedited.
- [ ] Red run observed and quoted; case 2 proves the map got the pre-cut list.
- [ ] `App.tsx` still has exactly one `composeView` call, one `WORLD`, and no coordinate maths.
- [ ] Lane suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the two files above plus this task file.
- [ ] `status: done`; `## Result` filled; one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
07-06: drive the results list from the map viewport

- points: 1
- red: test_bbox_around_bangkok_shows_only_buildings_in_view failed (Unable to find an element with the text: The Bangkok Sathorn)
- mount MapView in the bottom band and pass onViewportChange={setBBox}
- keep MapView on the filtered buildings and the list on the viewport-cut ones
- suite: N passed
```

Files in the commit: exactly the two listed files plus this task file.

## Traps and review notes

- Passing `inView` to the map instead of `buildings` silently removes bubbles as the map moves and makes 07-14's "two bubbles" assertion impossible; case 2 catches it.
- Cutting twice (once in `App`, once in `MapView`) doubles the work and can make the two halves disagree by one building at the edge.
- Wrapping `setBBox` in a debounce here duplicates 07-05 and makes the E2E budget unmeasurable.
- Re-deriving the lists inside JSX (`splitByLocation(...)` in the render body) re-cuts on every keystroke; the memo from 05-17 owns it.
- Editors of `App.tsx` in this stage: 07-06, 07-09, 07-10, 07-12. Run them one at a time; a stale branch of `App.tsx` is the most likely merge loss in this stage.
- Reviewers: grep `App.tsx` for `setBBox` (one), `MapView` (one), `Math.` (none in the viewport path).

## Plan samples are untrusted

The plan's integration step is one sentence ("`App` integration: with a bbox around Bangkok, `ResultsList` shows only buildings inside it"). Verify `useViewport`'s and `MapView`'s actual signatures in the repo before wiring (`CLAUDE.md` §6) and record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
