---
id: 07-04
stage: 07
title: moveend subscription, bounds and cleanup
points: 1
owner: web
status: todo
depends_on: [07-01, 05-15]
files:
  - web/src/components/MapEvents.tsx
  - web/src/components/MapView.tsx
  - web/src/components/MapView.test.tsx
---

# 07-04 · `moveend` subscription, bounds and cleanup

**Points:** 1 · **Stage:** 07 — Map and selection · **Owner lane:** web · **Depends on:** 07-01, 05-15 · **Unblocks:** 07-05

## Goal

Extract the map-event child into `MapEvents`, have it report the map's current bounds as a `BBox` on `moveend`, and unsubscribe both of its listeners on unmount — the map instance staying in a ref, never in React state.

## Why this task exists

The right-hand list must follow the map ("when the map moves, update the list on the right", `plan/02-architecture.md` §5.1), and the only event that means "the user finished moving" is `moveend` (F8). That conversion — Leaflet's `LatLngBounds` into the app's `BBox` — has to exist exactly once, in one component, with a cleanup path: a leaked `moveend` listener keeps firing into an unmounted tree, and a re-subscription on every render turns one drag into a dozen state writes. This task is the plumbing only; the throttle and the list wiring are 07-05 and 07-06.

## Current state (preconditions)

- 07-03 done: `MapView.tsx` contains a module-private `MapEvents` that calls `useMap()`, subscribes `map.on("click", …)` for deselection and unsubscribes on unmount; `MapView.test.tsx` mocks `react-leaflet` including `useMap`.
- 07-01 done: `MapView` renders one `MapContainer`; `web/src/test/setup.ts` carries the layout stubs.
- 05-15 done: `web/src/state/useViewport.ts` returns `{ bbox, setBBox }` with a stable setter and `null` meaning "whole city"; 05-11 done: `BBox` is `{ south, west, north, east }` in `web/src/lib/viewport.ts`.
- Nothing in the app calls `setBBox` yet — `App.tsx` passes no `onViewportChange` consumer (07-06 wires it).

## Read before you start

- `CLAUDE.md` §3, §5 (one implementation per rule; no `any` in `state/`), §6.
- `plan/02-architecture.md` §5.2 row F8 — `moveend` only, update within 200 ms, `zoomstart`/`movestart` do not count; §3.3 for the dataset's coordinate field names.
- `plan/stage-07-map.md` Task 7.2 items 1–2 and the Traps entry "do not store `map` in React state".
- `plan/90-test-matrix.md` row W-08 — its component half runs against this file.
- The installed Leaflet API: `map.on(name, fn)`, `map.off(name, fn)`, `map.getBounds()`, `LatLngBounds.getSouthWest()`/`getNorthEast()` and `LatLng.lat`/`.lng` — verify before asserting.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `web/src/components/MapEvents.tsx` | the map-event child, bounds conversion, cleanup |
| modify | `web/src/components/MapView.tsx` | delete the inline child, render the new one |
| modify | `web/src/components/MapView.test.tsx` | the six cases below |

Nothing else may change: no `App.tsx`, no `useViewport`, no `lib/` edit, no throttle (07-05).

## Test first (red)

The mocked `useMap` returns a fake map with `on`, `off` and a `getBounds` spy that returns a fake bounds object exposing `getSouthWest()` and `getNorthEast()`.

1. `test_subscribes_to_moveend_on_mount` — after rendering, `map.on` was called with `"moveend"` and a function, exactly once.
2. `test_moveend_reports_the_bounds_as_a_bbox` — the fake bounds report south-west `13.70 / 100.50` and north-east `13.80 / 100.60`; invoking the registered handler calls `onViewportChange` once with exactly `{ south: 13.7, west: 100.5, north: 13.8, east: 100.6 }` (four numbers, no Leaflet objects).
3. `test_unmount_removes_both_listeners` — after unmount, `map.off` was called for `"moveend"` and `"click"`, each with the same function reference that was registered.
4. `test_bounds_are_read_at_event_time_not_at_mount` — the fake bounds change between two handler invocations; the second call reports the new numbers (a cached bounds object would report the first pair twice).
5. `test_rerender_does_not_resubscribe_and_calls_the_latest_callback` — re-render with a new `onViewportChange` function: `map.on` was still called once and `off` not called; invoking the handler calls the **new** function (the handler reads the latest callback from a ref).
6. `test_map_click_still_deselects_after_the_extraction` — the 07-03 behaviour survives the move: the registered `"click"` handler calls `onSelect(null)`.

Expected red: `Failed to resolve import "./MapEvents"`, then `expected "moveend" to be called once`. Run `pnpm -C web test --run MapView`, quote one line.

## Implement (green)

Create `web/src/components/MapEvents.tsx`:

- Props exactly two: `onSelect` (a callback taking an id or `null`) and `onViewportChange` (a callback taking a `BBox`). The component renders nothing, i.e. it returns no element.
- `const map = useMap()`; keep the map in a ref and the two callbacks in refs updated on every render, so the subscription effect can run once with `[]` dependencies. No `useState` for the map or the bounds.
- One effect subscribes to the map's `click` and `moveend` events; its cleanup calls `off` for both, with the identical function references that were registered.
- `moveHandler` calls `map.getBounds()` when the event fires, reads `getSouthWest()`/`getNorthEast()` and calls `onViewportChange({ south, west, north, east })` — plain numbers, no `LatLng` leakage, no rounding, no padding.
- No `move`, `zoom`, `zoomstart`, `movestart`, `dragend` or `resize` subscription: `moveend` is the contract.
- One-sentence comment: "the only map-event child; click deselects, moveend reports the bounds".

In `MapView.tsx`: delete the inline `MapEvents` from 07-03, import the new module and render it inside the `MapContainer` next to the tile layer and markers, passing the two callbacks. The rendered behaviour must not change, so 07-01…07-03 cases stay green unedited.

Run: `pnpm -C web test --run MapView` — six new green, thirteen total in this file.

## Refactor / keep it DRY

One map-event child, one bounds conversion, one subscription lifetime. `MapView` must no longer mention `useMap`, `map.on` or `getBounds`; `MapEvents` must not know about markers, tiles or `App`. `BBox` is imported from `lib/viewport.ts` — no second `{south, west, north, east}` inline type. 07-05 adds coalescing inside `moveHandler`, not in `App`.

## Verification (how to test it)

```text
pnpm -C web test --run MapView
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Manual check: after 07-06, `pnpm -C web dev` → dragging the map visibly narrows the right-hand list; a zoom-in also narrows it (that is a `moveend` too); no console errors and no repeated state writes while dragging.

## Out of scope (do not do)

- Coalescing/throttling and the `fitBounds` feedback guard — 07-05.
- Wiring `setBBox` into `App` or changing `useViewport` — 07-06.
- `resize` handling, `invalidateSize` calls, `maxBounds`, `worldCopyJump`, a `bounds` prop on `MapContainer`.
- Storing the map, bounds or bbox in `useState`; a module-level map singleton.

## Definition of done

- [ ] All six cases exist, named as written, and pass; the 07-01…07-03 cases still pass unedited.
- [ ] Red run observed and quoted; cleanup identity and "bounds read at event time" are asserted.
- [ ] `MapView.tsx` contains no `useMap`/`getBounds`; `MapEvents.tsx` contains no state.
- [ ] Lane suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the three files above plus this task file.
- [ ] `status: done`; `## Result` filled; one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
07-04: report map bounds on moveend

- points: 1
- red: test_moveend_reports_the_bounds_as_a_bbox failed (expected "moveend" to be called once)
- extract MapEvents from MapView and keep the map in a ref, never in state
- subscribe moveend only, convert getBounds to {south, west, north, east}
- unsubscribe click and moveend on unmount with the same handler references
- suite: N passed
```

Files in the commit: exactly the three listed files plus this task file.

## Traps and review notes

- An inline arrow in the effect and a fresh arrow in cleanup is the leak this task exists to prevent; case 3 compares references.
- Caching `getBounds()` at mount makes the list freeze at the first viewport; case 4 fails.
- Subscribing to `move` or `zoom` "to be safe" breaks F8 and floods the list during a drag; the effect must name `moveend` only.
- Putting `[onViewportChange]` in the dependency array resubscribes on every parent render (the parent re-renders on every bbox change — a loop waiting to happen); the ref indirection exists for this.
- `MapContainer`'s `whenReady` is not a substitute: it gives no unsubscribe path and no event semantics.
- Reviewers: grep `web/src` for `getBounds` (one hit, in `MapEvents.tsx`) and for `useState` inside that file (none).

## Plan samples are untrusted

The plan describes `MapEvents` in prose and shows no subscription code. Verify Leaflet's `on`/`off`/`getBounds` and `LatLngBounds` accessors in the installed version, and `useMap`'s export from `react-leaflet@4.2.1` (`CLAUDE.md` §6). If the installed API differs, follow it and record the deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
