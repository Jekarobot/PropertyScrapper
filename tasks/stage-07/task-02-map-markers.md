---
id: 07-02
stage: 07
title: One bubble per building, radius and colour
points: 2
owner: web
status: todo
depends_on: [07-01]
files:
  - web/src/components/MapView.tsx
  - web/src/components/MapView.test.tsx
---

# 07-02 · One bubble per building, radius and colour

**Points:** 2 · **Stage:** 07 — Map and selection · **Owner lane:** web · **Depends on:** 07-01 · **Unblocks:** 07-03, 07-10

## Goal

Draw one bubble per mapped building at its coordinates, sized from the building's unit count by the F7 formula and coloured by the single accent pair, with the selected bubble in the darker accent.

## Why this task exists

The map is only useful if it answers "which building is that, and how big is it" without a click. F7 (`plan/02-architecture.md` §5.2) fixes the encoding: radius `8 + 4·sqrt(n)` px clamped to 8–22, so a 1-unit building is a small dot and a 20-unit building stops growing before it swallows the district; colour is one accent, `#0f766e` selected and `#14b8a6` at 80 % idle. These are the numbers the design review checks (W-07), and the E2E tasks count bubbles in the DOM, so the marker set must be exactly "the buildings with coordinates" — one each, none for the unmapped building, never a duplicate after a re-render.

## Current state (preconditions)

- 07-01 done: `MapView.tsx` renders one `MapContainer` with the CARTO `TileLayer`, accepts `{ buildings, selectedId, onSelect, onViewportChange, initialCenter? }`, and `MapView.test.tsx` mocks `react-leaflet`.
- 07-01 done: `web/src/test/setup.ts` has the `ResizeObserver`, `matchMedia` and `getBoundingClientRect` stubs.
- 05-03 done: the sample fixture has three buildings — `p00348` `The Bangkok Sathorn` at `13.7187616 / 100.5202356` with 3 units, `premier-thonglor` with 2 units and coordinates, `no-location` with `lat`/`lng` `null` and 1 unit.
- 07-01's six-way mock of `react-leaflet` exists; this task extends it with `CircleMarker` (the mock must record its props, not render Leaflet).

## Read before you start

- `CLAUDE.md` §3 (test-first, exact values), §5 (one implementation per rule; no `any`), §6.
- `plan/02-architecture.md` §5.2 row F7 — radius formula, clamp, selected/darker accent; §5.3 — accent `#0f766e`, idle `#14b8a6` at 80 % opacity.
- `plan/stage-07-map.md` Task 7.1 items 2, 3 and the "Leaflet in jsdom" rule.
- `plan/90-test-matrix.md` row W-07 — bubble radius/colour rules.
- `web/src/types.ts` — `Building.lat`/`lng` are `number | null`; `unitCount` is the recomputed count, not a stored total.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/components/MapView.tsx` | the bubble layer, radius rule, colours, marker classes |
| modify | `web/src/components/MapView.test.tsx` | the eight cases below |

Nothing else may change: no tooltips or click handlers (07-03), no legend (07-10), no `lib/` edit, no fixture edit.

## Test first (red)

The mocked `CircleMarker` records `center`, `radius`, `pathOptions` and `eventHandlers`. Build buildings by spreading a fixture building and overriding only `unitCount` (never invent coordinates).

1. `test_one_marker_per_building_with_coordinates` — the fixture's three buildings produce exactly two markers; `p00348`'s name and count appear once each (no duplicate marker for one building).
2. `test_null_coordinate_building_produces_no_marker` — only `no-location` in `buildings` → zero markers, and the map still renders.
3. `test_marker_center_is_latitude_then_longitude` — `p00348`'s marker receives `center` exactly `[13.7187616, 100.5202356]`; swapped order (`[100.5202356, 13.7187616]`) would put the building in the Indian Ocean, so the literal pair is the assertion.
4. `test_radius_follows_the_unit_count_formula_and_clamp` — table: `unitCount` 0 → 8 (clamped low), 1 → 12, 3 → 14.928…, 5 → 16.944…, 20 → 22 (clamped high). Assert the two irrational values with `toBeCloseTo(…, 3)` and the clamped ones exactly; the formula is `8 + 4·sqrt(n)` with no integer rounding.
5. `test_idle_bubble_is_teal_at_eighty_percent_opacity` — an unselected marker's `pathOptions.fillColor` is `#14b8a6` and `fillOpacity` is `0.8` (both assertions; the colour alone would pass while a solid bubble hides the tiles).
6. `test_selected_bubble_uses_the_darker_accent` — with `selectedId: "p00348"` that marker's `fillColor` is `#0f766e` and the other marker stays `#14b8a6`; exactly one marker is selected.
7. `test_each_marker_carries_a_stable_bubble_class` — every marker's `pathOptions.className` contains the token `bubble` and the building-specific token `bubble-<id>` (`bubble-p00348`). These are the only stable hooks the Playwright tasks (07-14…07-16) may count and click; do not rely on Leaflet's own class names.
8. `test_rerender_does_not_duplicate_markers` — re-render with the same buildings → still two markers; two buildings sharing identical coordinates → two markers (markers are keyed by building id, never deduped by position).

Expected red: `TypeError: cannot read properties of undefined` on `CircleMarker` from the mock, or `expected 0 to be 2`. Run `pnpm -C web test --run MapView`, quote one line.

## Implement (green)

- Render the markers inside the existing `MapContainer`, one `CircleMarker` per building whose `lat` and `lng` are both non-null, keyed by `building.id`. Skip a building when either coordinate is `null` — test with `== null` so a real `0` is not treated as missing.
- `center` is `[building.lat, building.lng]` in that order. No swapping, no rounding, no projection.
- Radius: one module-private function of `unitCount` implementing `8 + 4·sqrt(n)` clamped to `[8, 22]`; `unitCount` comes from the prop the map received and is never recomputed here.
- `pathOptions`: idle `fillColor` `#14b8a6`, `fillOpacity` 0.8, selected `fillColor` `#0f766e`; a thin stroke in the same colour is acceptable but must not change the two asserted values. Take both hex values from the Tailwind theme tokens where the token exists; never a third teal.
- `pathOptions.className` carries `bubble bubble-<id>` — the E2E contract.
- No tooltip, no `eventHandlers` yet (07-03), no ref, no state, no `useMap`. Do not import `splitByLocation` or `inBBox` here: the null check is a type narrowing for rendering, not a second viewport rule.

Run: `pnpm -C web test --run MapView` — eight green.

## Refactor / keep it DRY

One radius function, one colour pair, one null-coordinate predicate in this file. The formula exists nowhere else; `App` and the legend (07-10) reuse the rendered result, they never recompute a radius for a label. If the colour pair already exists as theme tokens, import them; if they exist only as literals in `BuildingRow`, extract the pair to the token module in this commit and say so in `## Decisions` — do not keep two hex tables.

## Verification (how to test it)

```text
pnpm -C web test --run MapView
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Manual check: `pnpm -C web dev` → two teal bubbles over Sathorn and Thonglor; a short building is visibly smaller than a long one; clicking one (after 07-03) darkens it; the unmapped building has no bubble.

## Out of scope (do not do)

- Tooltips, click handlers, hover states, keyboard interaction — 07-03.
- Selection state: the marker reads `selectedId` from props; it never stores selection.
- Clustering, spiderfying, marker images, popups, a bubble for the no-location building, a separate "you are here" marker.
- Legend, dot-size keys, counts — 07-10; zoom options — 07-11.
- Changing the fixture or adding buildings so the radius table is easier to write.

## Definition of done

- [ ] All eight cases exist, named as written, and pass; `test_radius_follows_the_unit_count_formula_and_clamp` keeps its table.
- [ ] Red run observed and quoted; the 1/5/20 radius values and both `fillColor`/`fillOpacity` pairs are asserted.
- [ ] No marker for a null-coordinate building, and no second marker for one building after a re-render.
- [ ] Lane suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the two files above plus this task file.
- [ ] `status: done`; `## Result` filled; one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
07-02: draw one bubble per mapped building

- points: 2
- red: test_one_marker_per_building_with_coordinates failed (expected 0 to be 2)
- add CircleMarker per building with coordinates, skipping null coordinates
- encode unit count as radius 8 + 4*sqrt(n) clamped to 8..22
- colour idle #14b8a6 at 0.8 fill opacity, selected #0f766e, class bubble bubble-<id>
- suite: N passed
```

Files in the commit: exactly the two listed files plus this task file.

## Traps and review notes

- `lat`/`lng` order is the classic bug: case 3 uses the fixture's literal pair, and the reviewed code must read `[lat, lng]`.
- Rounding the radius to an integer passes a visual check and fails the table; the formula is asserted at three decimals.
- Clamping only the top fails case 4's `0 → 8`; clamp both ends even though a 0-unit building never reaches production.
- A bubble for the null-coordinate building would contradict F6 and break 07-14's "two bubbles" assertion.
- Deduping markers by position would hide two real buildings that share a tower; case 8 pins one marker per id.
- Without `bubble-<id>` the E2E tasks would have to click markers by pixel position and would be flaky by construction.
- Reviewers: grep `MapView.tsx` for `Math.sqrt` (one hit), `fillColor` (one pair), and for `0f766e`/`14b8a6` appearing twice.

## Plan samples are untrusted

The plan's props sketch and the F7 sentence are illustrative. Verify `CircleMarker`'s prop names (`center`, `radius`, `pathOptions`, `eventHandlers`) and Leaflet's `PathOptions` field names (`fillColor`, `fillOpacity`, `className`) against the installed `react-leaflet@4.2.1`/`leaflet@1.9.4` types (`CLAUDE.md` §6). Record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
