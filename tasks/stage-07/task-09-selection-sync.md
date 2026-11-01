---
id: 07-09
stage: 07
title: One selection across map, list and panel
points: 1
owner: web
status: todo
depends_on: [07-06, 07-03, 07-08]
files:
  - web/src/App.tsx
  - web/src/App.test.tsx
---

# 07-09 · One selection across map, list and panel

**Points:** 1 · **Stage:** 07 — Map and selection · **Owner lane:** web · **Depends on:** 07-06, 07-03, 07-08 · **Unblocks:** 07-12, 07-13

## Goal

Wire the one `selectedId` from `App` into the map, the rows and the panel so a bubble click, a row click and a close all produce the same state — and a selection that filtering removes is cleared instead of leaving a ghost panel.

## Why this task exists

Selection is the app's spine: F9 (`plan/02-architecture.md` §5.2) says selecting a building sets `selectedId` and deselecting clears the panel, and W-09 is the matrix row that proves all four directions. The pieces exist — `useSelection` (05-16) with its stale-selection cleanup, `BuildingRow` with `aria-current`, `MapView` with `selectedId`, `BuildingPanel` with a close — but nothing has ever proven they agree. Two independent selection states is the classic failure: the bubble highlights, the panel shows another building, and closing one leaves the other behind. This is the last wiring task before the E2E stage, and the behaviour it proves is what a reviewer demonstrates by hand.

## Current state (preconditions)

- 07-06 done: `App` mounts `MapView` with `{ buildings, selectedId, onSelect, onViewportChange }` and renders `ResultsList` from `inView`; `App.test.tsx` mocks `MapView` so a case can invoke its props.
- 07-03 done: a marker click calls `onSelect(id)`; a map click calls `onSelect(null)`.
- 07-08 done: `BuildingPanel` renders the selected building, its units and the `Clear selection` control; its own cases are green.
- 05-16 done: `useSelection(buildings)` owns `selectedId`, `select(id | null)` and clears the id when it is no longer among the given buildings; `App` already calls it.
- `App` renders the panel **only** if 07-09 adds it: the layout has the left column, but `BuildingPanel` is not mounted there yet.

## Read before you start

- `CLAUDE.md` §3, §5 (single source of truth: selection lives once, in the hook), §6.
- `plan/02-architecture.md` §5.1 — "selecting a building (bubble or row) fills the left panel and highlights the bubble"; §5.2 row F9; F10 (state in React only, no URL sync).
- `plan/stage-07-map.md` Task 7.3 item 7 — the full selection-sync checklist.
- `plan/90-test-matrix.md` row W-09 — this task is its integration proof.
- `web/src/state/useSelection.ts`, `web/src/components/BuildingRow.tsx`, `web/src/components/MapView.tsx`, `web/src/components/BuildingPanel.tsx` — the four consumers.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/App.tsx` | mount the panel, pass one `selectedId` everywhere |
| modify | `web/src/App.test.tsx` | the six cases below |

Nothing else may change: no component internals, no hook, no `lib/` edit, no CSS. If a component cannot express something the tests need, report it instead of editing it here.

## Test first (red)

`MapView` is mocked; a case reads the latest props and invokes `onSelect` inside `act`. Real `ResultsList`, `BuildingRow` and `BuildingPanel` are rendered.

1. `test_marker_click_selects_the_building_and_renders_the_panel` — invoke the map's `onSelect("p00348")`: the panel shows `The Bangkok Sathorn` and its three units, and the row for that building carries `aria-current="true"`.
2. `test_the_map_receives_the_selected_id` — after the same click, the newest `MapView` props carry `selectedId: "p00348"` (the darker bubble itself is 07-02's prop assertion; here the app must pass the id).
3. `test_row_click_updates_the_map_and_the_panel` — click the `Premier Thonglor` row: the newest `MapView` props carry `selectedId: "premier-thonglor"`, that row alone has `aria-current="true"`, and the panel shows the new name (the previous building's units are gone).
4. `test_clear_selection_empties_panel_map_and_rows` — with a selection, click `Clear selection`: the panel shows the placeholder instruction, `MapView` receives `selectedId: null`, and no row has `aria-current`.
5. `test_clicking_the_empty_map_clears_the_selection` — invoke `onSelect(null)` with a selection: same three effects as case 4, i.e. both deselect paths share one implementation.
6. `test_filter_change_that_removes_the_selected_building_clears_the_selection` — select `p00348`, then set `district` to `Khlong Toei` (the only remaining building is `no-location`): the panel shows the placeholder, no row has `aria-current`, and `MapView` receives `selectedId: null` — the ghost-panel case from 05-16, proven end to end.

Expected red: `Unable to find an element with the text: The Bangkok Sathorn` in the left column after the click (the panel is not mounted) and `expected undefined to be "p00348"` for the prop case. Run `pnpm -C web test --run App`, quote one line.

## Implement (green)

- `App` passes the hook's single `selectedId` to all three consumers: `MapView` (`selectedId`), `ResultsList` (`selectedId`) and `BuildingPanel` (the resolved building). The panel's building comes from the same grouped list the rows use — `useSelection` may expose the resolved object or `App` finds it; do not add a second `find` that can disagree with the hook (05-16 owns resolution).
- `onSelect={select}` is passed to `MapView` and to the list; there is exactly one selection setter in the tree.
- The panel mounts in the left column (the layout's placeholder is replaced). `onClose={…}` calls `select(null)` — the same function the map's empty click calls, not a second "clear" path.
- Deselect has exactly one meaning (`null`); never `undefined`, `""`, or "the first building".
- No URL change, no hash write, no `localStorage`: F10.
- One comment above the panel mount: "one selection: bubble → row → panel → deselect".

Run: `pnpm -C web test --run App` — six new green, the 05-17/06-05/06-09/06-12/07-06 cases still green.

## Refactor / keep it DRY

One `selectedId`, one setter, one resolution of the selected building, one close path. `BuildingRow`, `MapView` and `BuildingPanel` stay dumb: they read a prop and call a callback. If `App` ends up with two `select`-shaped functions or a `clearSelection` wrapper, delete one. The earlier stage-05 placeholder panel markup must be removed, not left beside the real one.

## Verification (how to test it)

```text
pnpm -C web test --run App
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Manual check: `pnpm -C web dev` → click a bubble: the left panel fills and the bubble darkens; click a row: the panel switches and the bubble moves; the ×, an empty-map click and the no-location row all clear consistently; filtering the selected building out empties the panel.

## Out of scope (do not do)

- The no-location bucket markup and the legend — 07-10; the mobile toggle — 07-12.
- URL/hash sync of the selection, deep links, `history.pushState`, scroll-into-view, focus management after selecting (08-07 owns a11y polish).
- Multi-select, hover preview, "compare" mode, a state library.
- Editing `useSelection`, `BuildingRow`, `MapView` or `BuildingPanel`.

## Definition of done

- [ ] All six cases exist, named as written, and pass; every earlier `App` case still passes unedited.
- [ ] Red run observed and quoted; all four directions (bubble, row, close, map click) plus the stale-filter case are covered.
- [ ] Exactly one `select`/`selectedId` source in `App.tsx`; the panel is mounted, and no stage-05 placeholder panel remains.
- [ ] Lane suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the two files above plus this task file.
- [ ] `status: done`; `## Result` filled; one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
07-09: sync selection across map, list and panel

- points: 1
- red: test_marker_click_selects_the_building_and_renders_the_panel failed (Unable to find the text: The Bangkok Sathorn)
- mount BuildingPanel and pass the hook's single selectedId to map, list and panel
- close, empty-map click and the stale-filter cleanup share one deselect path
- suite: N passed
```

Files in the commit: exactly the two listed files plus this task file.

## Traps and review notes

- A second selection state in the panel (or a local "last selected building") produces a ghost panel after filtering; case 6 is the guard.
- A `clearSelection` wrapper next to `select(null)` splits the deselect path and drifts; the tests call both and expect identical effects.
- Passing `undefined` instead of `null` makes `MapView`'s prop check pass while the row keeps `aria-current`; assert `null` exactly.
- Leaving the stage-05 placeholder panel mounted doubles the left column and breaks the "one heading" expectation of 07-07's cases.
- `App.tsx` is edited by 07-06, 07-09, 07-10 and 07-12: never run two of them concurrently.
- Reviewers: grep `App.tsx` for `select(` (one callback), `useState` around selection (none — the hook owns it), and `location.hash` (none).

## Plan samples are untrusted

The plan lists the sync bullets without code. Read the actual `useSelection` signature and the components' props in the repo (`CLAUDE.md` §6) — 05-16's hook may expose the id and a setter, or a resolved building; follow the repo and record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
