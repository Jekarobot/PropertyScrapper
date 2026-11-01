---
id: 07-03
stage: 07
title: Marker click, map click and tooltips
points: 1
owner: web
status: todo
depends_on: [07-02]
files:
  - web/src/components/MapView.tsx
  - web/src/components/MapView.test.tsx
---

# 07-03 · Marker click, map click and tooltips

**Points:** 1 · **Stage:** 07 — Map and selection · **Owner lane:** web · **Depends on:** 07-02 · **Unblocks:** 07-09

## Goal

Make the bubbles selectable: a marker click reports its building id without also reaching the map, a click on empty map reports "no selection", and every marker carries a tooltip with its name, unit count and price range.

## Why this task exists

Rule F9 (`plan/02-architecture.md` §5.2) has two halves — selecting a building and deselecting it — and on the map both are clicks. If the marker click bubbles to the map, every selection is immediately cleared again; if a map click does not clear, there is no way back to the "select a building" placeholder. The tooltip is what makes a bubble readable before committing to a click, and it is the only place on the map that shows the recomputed price range. This task is the map half of W-09; the list and panel halves are 07-08/07-09.

## Current state (preconditions)

- 07-01/07-02 done: `MapView` renders one `MapContainer` with the tile layer and one `CircleMarker` per mapped building with `radius`, `pathOptions` and the `bubble bubble-<id>` classes.
- `MapView.test.tsx` mocks `react-leaflet`, records marker props, and mocks `useMap` where a test needs a fake map instance.
- 05-10 done: `lib/group.ts` exports `countLabel(n)` (`3 units`) and `priceRange(building)` (`45–50k`); 05-12 done: `lib/format.ts`.
- No marker `eventHandlers`, no `Tooltip`, and no map-level click subscription exist yet.

## Read before you start

- `CLAUDE.md` §3, §5 (DRY: `countLabel`/`priceRange` are imported, never re-implemented), §6.
- `plan/02-architecture.md` §5.2 row F9 — select and deselect semantics; §5.1 — the bubble tooltip content.
- `plan/stage-07-map.md` Task 7.1 items 4 and 5, and the Traps entry about the map instance living in a ref.
- `plan/90-test-matrix.md` row W-09 (selection sync) — the marker half of it is proven here.
- The installed `react-leaflet@4.2.1` API: `<MapContainer>` has no `eventHandlers` prop, so the map-level click must come from a child component that calls `useMap()` — verify this against `node_modules/react-leaflet` before coding.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/components/MapView.tsx` | marker `eventHandlers`, tooltips, the map-click child |
| modify | `web/src/components/MapView.test.tsx` | the five cases below |

Nothing else may change: no `App.tsx`, no panel, no `lib/` edit, no legend, no `moveend` (07-04).

## Test first (red)

1. `test_marker_click_selects_the_building_and_stops_propagation` — invoke the marker's `eventHandlers.click` with a fake Leaflet event whose `originalEvent.stopPropagation` is a spy: `onSelect` is called exactly once with `p00348`, the spy is called once, and `onSelect` is never called with `null` in the same interaction (otherwise the map's own click would immediately deselect).
2. `test_map_click_clears_the_selection` — the component subscribes to the map instance's `click` (assert `map.on` was called with `"click"` and a function); invoking that function calls `onSelect(null)` once.
3. `test_unmount_removes_the_map_click_listener` — after unmount, `map.off` was called with `"click"` and the **same** handler reference that was registered (identity, not a new function).
4. `test_each_marker_has_one_tooltip_with_name_count_and_price_range` — `p00348`'s tooltip contains `The Bangkok Sathorn`, `3 units` and `45–50k`; exactly one `Tooltip` per marker (three buildings with two coordinates → two tooltips).
5. `test_empty_buildings_render_no_markers_or_tooltips_and_the_map_still_clears` — `buildings: []` → no marker, no tooltip, map click still calls `onSelect(null)`; no throw.

Expected red: `expected [] to have length 1` for the tooltip case, and `expected "click" to be called` for the map-click case. Run `pnpm -C web test --run MapView`, quote one line.

## Implement (green)

- Marker: `eventHandlers.click` receives the Leaflet mouse event, stops it from reaching the map (call `stopPropagation` on the event's `originalEvent`, after verifying that shape in the installed types), then calls `onSelect(building.id)` once. No hover handler, no popup, no keyboard handler.
- Tooltip: one `Tooltip` child per marker with the building name, `countLabel(building.unitCount)` and `priceRange(building)` — imported from `lib/group.ts`, never re-formatted inline. No markup beyond what those three strings need.
- Map click: a module-private child named `MapEvents` (defined in the same file for now) that calls `useMap()`, subscribes with `map.on("click", handler)` in an effect and removes it with `map.off("click", handler)` on unmount using the same reference. Keep the map in a `useRef`/closure, never in `useState`. 07-04 moves this component to its own module and adds `moveend`; name it `MapEvents` now and keep the click logic unchanged.
- `onSelect` is a prop: the component holds no selection state and never calls `onViewportChange` from a click.

Run: `pnpm -C web test --run MapView` — five green, the seven 07-01/07-02 cases still green.

## Refactor / keep it DRY

Two event handlers, one selection contract: the marker reports an id, the map reports `null`. Do not add a "clear selection" callback: `onSelect(null)` is the one way to deselect. `MapEvents` is the single home of map-level subscriptions — 07-04 and 07-05 extend it rather than adding a second child that also calls `useMap()`. `countLabel`/`priceRange` are imported; a local `units` pluraliser or a price formatter in the tooltip is a duplicate and must be deleted.

## Verification (how to test it)

```text
pnpm -C web test --run MapView
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Manual check: `pnpm -C web dev` → hovering a bubble shows its card; clicking a bubble selects it once (the selection is not immediately cleared); clicking empty map space clears it; no console errors.

## Out of scope (do not do)

- `moveend`, viewport bounds, throttling — 07-04/07-05.
- The building panel, row highlighting, the no-location bucket — 07-07…07-10.
- Popups (`Popup`), clustering, marker hover previews, tooltips on the map itself, keyboard navigation of markers (08-07).
- A second map-event child or a map instance kept in React state.

## Definition of done

- [ ] All five cases exist, named as written, and pass; all earlier `MapView` cases still pass unedited.
- [ ] Red run observed and quoted; the propagation assertion is a spy assertion, not a comment.
- [ ] Marker click does not produce a `null` selection and map click does, proven in the same file.
- [ ] Lane suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the two files above plus this task file.
- [ ] `status: done`; `## Result` filled; one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
07-03: make bubbles selectable and add tooltips

- points: 1
- red: test_map_click_clears_the_selection failed (expected "click" to be called)
- add marker click that selects the building and stops propagation to the map
- add a map-click child using useMap that deselects and unsubscribes on unmount
- add one tooltip per marker with name, countLabel and priceRange
- suite: N passed
```

Files in the commit: exactly the two listed files plus this task file.

## Traps and review notes

- Without stopping propagation, Leaflet's map click fires on the same gesture and every selection is cleared a millisecond later; case 1 is the guard, and it fails loudly if the stop is dropped.
- Registering the map handler with an inline arrow and unsubscribing with a different arrow leaks the listener forever; case 3 asserts reference identity, not just that `off` was called.
- `MapContainer` does not accept an `eventHandlers` prop in `react-leaflet@4.2.1`; assuming it does produces a silently ignored prop. The `useMap()` child is the supported route (verify in `node_modules`).
- Re-implementing `priceRange` in the tooltip drifts from the row's string; the shared helper is asserted by exact value (`45–50k` with the en dash).
- Reviewers: grep `MapView.tsx` for `useMap` (one component), `onSelect(null)` (one call site) and `priceRange` (imported, once).

## Plan samples are untrusted

The plan names `MapEvents` and `useMap()` without showing the subscription. Verify Leaflet's event object shape (`originalEvent`, `stopPropagation`), `map.on`/`map.off` signatures and `Tooltip`'s children contract against the installed versions (`CLAUDE.md` §6). A sample that puts `eventHandlers` on `MapContainer` is stale — follow the library and record it under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
