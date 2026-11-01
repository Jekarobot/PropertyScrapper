---
id: 07-11
stage: 07
title: Zoom control placement and wheel policy
points: 0.5
owner: web
status: todo
depends_on: [07-01]
files:
  - web/src/components/MapView.tsx
  - web/src/components/MapView.test.tsx
---

# 07-11 · Zoom control placement and wheel policy

**Points:** 0.5 · **Stage:** 07 — Map and selection · **Owner lane:** web · **Depends on:** 07-01 · **Unblocks:** nothing

## Goal

Move the zoom control to the bottom right, disable wheel zoom and keep double-click zoom on, so the bottom map never steals the page's scroll while remaining zoomable from the buttons and a double click.

## Why this task exists

The map is a full-width band at the bottom of the viewport (`plan/02-architecture.md` §5.1), so a wheel over it must scroll the page, not zoom the map — otherwise the page feels broken on a laptop trackpad (the plan's Traps). A user who wants to zoom still needs an affordance: Leaflet's buttons and double click. The default zoom control sits top-left, which is exactly where the 340 px left panel and the map's own top edge meet; the bottom-right corner is free. The legend (07-10) explains the wheel policy in words; this task makes the behaviour match the words.

## Current state (preconditions)

- 07-01 done: `MapView` renders one `MapContainer` with the CARTO `TileLayer` and the props contract; `MapView.test.tsx` mocks `react-leaflet`.
- 07-10 done by lane order: `MapLegend` renders the scroll hint `Use ⌘/Ctrl + scroll to zoom` and is mounted inside the map container. This task must not duplicate that hint.
- The map currently uses Leaflet's defaults: zoom control top-left, wheel zoom on, double-click zoom on.

## Read before you start

- `CLAUDE.md` §3 (0.5 means one mechanical change plus its direct test), §6.
- `plan/02-architecture.md` §5.2 row F8 and the map paragraph in §5.1; the Traps entry "`scrollWheelZoom` must be off, otherwise the bottom map steals page scrolling".
- `plan/stage-07-map.md` Task 7.4 items 3–4.
- `plan/90-test-matrix.md` row W-08 (scroll behaviour is measured end to end in 07-14; this task pins the options).
- The installed `react-leaflet@4.2.1`: `MapContainer`'s `zoomControl` is a boolean, and the position lives on the exported `ZoomControl` component — verify in `node_modules/react-leaflet` before writing the assertion.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/components/MapView.tsx` | the three map options and the positioned zoom control |
| modify | `web/src/components/MapView.test.tsx` | the three cases below |

Nothing else may change: no legend edit, no CSS, no new dependency, no `App.tsx`.

## Test first (red)

1. `test_zoom_control_is_placed_bottom_right` — `MapContainer` receives `zoomControl` `false` (Leaflet's default control is disabled because it cannot be positioned) and exactly one `ZoomControl` child is rendered with `position` `"bottomright"`. Assert both: the boolean alone would leave no control, the child alone would leave two.
2. `test_scroll_wheel_zoom_is_disabled_and_double_click_zoom_is_enabled` — `MapContainer` receives `scrollWheelZoom` `false` and `doubleClickZoom` `true` (explicit, not left to the default).
3. `test_no_other_map_option_is_changed_and_the_hint_appears_once` — the options this component passes are exactly the documented set (center, zoom, zoomControl, scrollWheelZoom, doubleClickZoom and the class name); `dragging`, `touchZoom`, `keyboard` and `attributionControl` are not overridden, and there is still exactly one `TileLayer` with the CARTO URL. The legend's hint `Use ⌘/Ctrl + scroll to zoom` (07-10) is rendered exactly once inside the map container — the policy and its explanation appear together, and no second copy lives in `MapView`.

Expected red: `expected undefined to be false` for `scrollWheelZoom` and `expected 0 to be 1` for the `ZoomControl` child. Run `pnpm -C web test --run MapView`, quote one line.

## Implement (green)

- Pass `zoomControl={false}` to `MapContainer` and render Leaflet's `ZoomControl` component as a child with `position="bottomright"`. Do not position it with CSS overrides.
- Pass `scrollWheelZoom={false}` and `doubleClickZoom={true}`.
- Change nothing else: dragging, touch zoom, keyboard handling and the attribution control keep their defaults; the tile URL and the `bubble` markers stay as they are.
- No legend text here: the hint belongs to `MapLegend` (07-10) and must appear exactly once in the tree.
- One comment above the options: the wheel is reserved for page scroll (F8), the buttons and double click remain.

Run: `pnpm -C web test --run MapView` — three green, the existing cases still green.

## Refactor / keep it DRY

The three options live in one place, next to the `MapContainer`, with no second `ZoomControl` and no `whenReady` option-juggling. The zoom limits, `minZoom`/`maxZoom` and the initial zoom remain 07-01's constants.

## Verification (how to test it)

```text
pnpm -C web test --run MapView
pnpm -C web test --run
pnpm -C web typecheck
make check
```

(`pnpm -C web e2e` from 07-13 onward re-proves the wheel behaviour against a real browser in 07-14.)

Manual check: `pnpm -C web dev` → the +/− control sits in the bottom-right corner of the map; a wheel over the map scrolls the page (or does nothing when the page cannot scroll) and never zooms; a double click zooms; the legend's hint is the only place the modifier is named.

## Out of scope (do not do)

- `minZoom`/`maxZoom`, a `ZoomControl` in a second position, a custom zoom slider, animated zoom options.
- The legend hint text (07-10) or a second copy of it.
- Touch gestures, `tap`, `inertia` tuning, `preferCanvas`.
- `scrollWheelZoom` on mobile-specific code (07-12) — the option is global.

## Definition of done

- [ ] All three cases exist, named as written, and pass; earlier `MapView` cases still pass unedited.
- [ ] Red run observed and quoted; both the `zoomControl` boolean and the `ZoomControl` position are asserted.
- [ ] `dragging`, `touchZoom`, `keyboard` and `attributionControl` are untouched (case 3).
- [ ] Lane suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the two files above plus this task file.
- [ ] `status: done`; `## Result` filled; one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
07-11: move the zoom control and disable wheel zoom

- points: 0.5
- red: test_scroll_wheel_zoom_is_disabled_and_double_click_zoom_is_enabled failed (expected undefined to be false)
- disable the default zoom control and render ZoomControl at bottomright
- disable scrollWheelZoom, keep doubleClickZoom, change nothing else
- suite: N passed
```

Files in the commit: exactly the two listed files plus this task file.

## Traps and review notes

- Passing `zoomControl={false}` without rendering `<ZoomControl>` leaves the map with no zoom affordance; passing `position` to `MapContainer` does nothing in `react-leaflet@4.2.1`.
- Leaving `scrollWheelZoom` undefined passes a "not false" assertion while wheel zoom stays on — assert `false`.
- `doubleClickZoom` must be explicit: a later refactor that spreads options could silently drop it.
- A second hint string in `MapView` duplicates 07-10's text; reviewers grep for `scroll to zoom` (one hit, in `MapLegend.tsx`).
- Do not "fix" the wheel by stopping propagation on the map container: that breaks page scroll in the other direction.

## Plan samples are untrusted

The plan states the intent (`zoomControl` at bottom right, wheel off, double click on) without showing props. Verify against the installed `react-leaflet@4.2.1` exports — `ZoomControl` may be exported from `react-leaflet` or only from its core package (`CLAUDE.md` §6). Follow the library and record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
