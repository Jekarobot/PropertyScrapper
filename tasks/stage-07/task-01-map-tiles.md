---
id: 07-01
stage: 07
title: Leaflet map with CARTO tiles and attribution
points: 2
owner: web
status: todo
depends_on: [05-01, 05-03]
files:
  - web/src/components/MapView.tsx
  - web/src/components/MapView.test.tsx
  - web/src/test/setup.ts
  - web/package.json
  - web/pnpm-lock.yaml
---

# 07-01 · Leaflet map with CARTO tiles and attribution

**Points:** 2 · **Stage:** 07 — Map and selection · **Owner lane:** web · **Depends on:** 05-01, 05-03 · **Unblocks:** 07-02, 07-04, 07-11

## Goal

Render the one Leaflet map — CARTO Positron tiles with their required attribution, the documented `MapView` props and a full-height container — so every later stage-07 task has a map to hang bubbles and events on.

## Why this task exists

The product's second half is "where are these buildings": the bottom map (`plan/02-architecture.md` §5.1) shows one bubble per building and drives the right-hand list. None of that is possible until one component owns the map instance, its tile layer and its licence text. The tile provider is a licence obligation, not a cosmetic choice — CARTO's terms require attribution to OpenStreetMap and CARTO on every map — and it is a test-matrix row (W-16). This task also installs the map stack the plan already chose, so later tasks never touch `package.json` again.

## Current state (preconditions)

- 05-01 done: the Vite/React/TS/Vitest scaffold, the six Tailwind tokens (including `accent`), `web/src/test/setup.ts` with jest-dom.
- 05-02 done: `setup.ts` has the no-op `ResizeObserver` Leaflet constructs under jsdom; `pnpm -C web test --run` is green.
- 05-03 done: `web/src/types.ts` with `Building` (`lat`/`lng` are `number | null`) and the 6-unit / 3-building sample fixture.
- 05-11 done by lane order: `web/src/lib/viewport.ts` exports `BBox` (`south`, `west`, `north`, `east`). `MapView` imports that type; if the file is absent, stop — do not define a second box shape.
- `leaflet`, `react-leaflet` and `@types/leaflet` are **not** installed yet. `web/src/components/` holds the stage-06 components; `MapView.tsx` and its test do not exist.

## Read before you start

- `CLAUDE.md` §3 (TDD; component tests never touch the network), §5 (pinned dependencies, single source of truth), §6 (verify the installed API before trusting a plan snippet).
- `plan/02-architecture.md` §4 — leaflet `1.9.4`, `react-leaflet` `4.2.1`, `@types/leaflet`; the Tile row naming `basemaps.cartocdn.com/light_all` and the OSM+CARTO attribution duty; the "Leaflet in jsdom" paragraph (component tests mock `react-leaflet` and assert props).
- `plan/02-architecture.md` §5.1 — the map is full width, 45 vh, sticky at the bottom; the container height comes from the layout, not from this component.
- `plan/stage-07-map.md` Task 7.1 — the props block, the six TDD items and the `setup.ts` stubs.
- `plan/90-test-matrix.md` row W-16 — tile URL and attribution must be asserted in a test.
- `web/src/types.ts` and `web/src/lib/viewport.ts` — the two types this task's props use.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `web/src/components/MapView.tsx` | the single map, tile layer, props, stylesheet import |
| create | `web/src/components/MapView.test.tsx` | the eight cases below, with `react-leaflet` mocked |
| modify | `web/src/test/setup.ts` | keep the 05-02 stub; add `matchMedia` and `getBoundingClientRect` |
| modify | `web/package.json` | the three pinned packages, nothing else |
| modify | `web/pnpm-lock.yaml` | the lockfile for those pins |

Nothing else may change: no `App.tsx`, no layout, no markers, no new scripts.

## Test first (red)

`react-leaflet` is mocked with `vi.mock` so every assertion is about the props this component passes, never about Leaflet internals. Read `web/src/test/fixtures/dataset.sample.json` for the buildings.

1. `test_renders_exactly_one_map_container` — rendering `MapView` mounts `MapContainer` exactly once; a re-render with the same props does not add a second one (two maps would be a second Leaflet instance and a licence violation).
2. `test_initial_center_and_zoom_default_to_bangkok` — `MapContainer` receives `center` `[13.7563, 100.5018]` and `zoom` `12`; passing `initialCenter` `[13.7, 100.5]` overrides only the center, zoom stays 12.
3. `test_tile_layer_uses_carto_positron` — the `TileLayer` child receives `url` exactly `https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png`; exactly one `TileLayer` is rendered.
4. `test_attribution_names_openstreetmap_and_carto` — the `attribution` string contains both `OpenStreetMap` and `CARTO` (case-exact substrings, both required) — W-16 and the licence.
5. `test_container_fills_its_parent` — the map root carries the `h-full` token (and the container element `w-full`); `MapView` adds no fixed pixel height, because the 45 vh belongs to the layout.
6. `test_empty_buildings_array_still_renders_the_map` — `buildings: []` → one `MapContainer`, no marker children, no throw.
7. `test_leaflet_stylesheet_is_imported` — read `MapView.tsx` as text: it imports `leaflet/dist/leaflet.css`. jsdom does not process CSS, so the import is asserted at the source and the visual result is the manual check.
8. `test_setup_provides_the_layout_stubs_leaflet_needs` — `window.matchMedia("(min-width: 768px)")` returns an object with a boolean `matches`, a `media` string and callable `addEventListener`/`removeEventListener`; `document.body.getBoundingClientRect()` reports `width` 800 and `height` 400; the 05-02 `ResizeObserver` still exists.

Expected red: `Failed to resolve import "./MapView"` for cases 1–6, and `window.matchMedia is not a function` for case 8. Run `pnpm -C web test --run MapView`, quote one failure line. A syntax error is not red — fix the test first.

## Implement (green)

Create `web/src/components/MapView.tsx`:

- Props exactly five: `buildings` (the filtered `Building` array), `selectedId` (a building id or `null`), `onSelect` (a callback taking an id or `null`), `onViewportChange` (a callback taking a `BBox`) and the optional `initialCenter` (a latitude/longitude pair). `selectedId`, `onSelect` and `onViewportChange` are accepted now and used by 07-02…07-05; do not leave them out and do not invent more props.
- Module constants: the default center `[13.7563, 100.5018]` and `zoom` 12, and the tile URL and attribution strings. One definition each — no component may repeat the URL.
- Import `leaflet/dist/leaflet.css` once, in this module. Import `leaflet` types only.
- Markup: one wrapper element carrying `relative h-full` and one `MapContainer` inside it with `h-full w-full`; the parent supplies the height (45 vh, layout task). No inline pixel height.
- `TileLayer` with the URL, the attribution string naming both OSM and CARTO, and nothing else configurable.
- No state, no effects, no `useMap` here: the map instance is not needed in this task.
- One comment above the component: it is the only place a Leaflet map is created, and the props are the whole contract with `App`.

Then `web/src/test/setup.ts`: keep the existing jest-dom import and `ResizeObserver`. Add `window.matchMedia` returning a `MediaQueryList`-shaped object whose default `matches` is `false` (desktop) with no-op `addEventListener`, `removeEventListener`, `addListener`, `removeListener` and `dispatchEvent`, and override `HTMLElement.prototype.getBoundingClientRect` to return 800×400 at zero offsets. One comment says Leaflet measures its container and jsdom reports zeroes; add nothing else (no IntersectionObserver, no scrollTo).

Run: `pnpm -C web test --run MapView` — eight green; then `pnpm -C web typecheck`.

## Refactor / keep it DRY

One map, one tile layer, one attribution string, one `BBox` type (imported from `lib/viewport.ts`). No Leaflet import outside this file and `MapEvents.tsx` (07-04). No second default center anywhere: 07-06 and the E2E harness read the map, they never re-declare coordinates. Delete any stub the test file added while debugging and keep the single `setup.ts` version.

## Verification (how to test it)

```text
pnpm -C web test --run MapView
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Manual check: `pnpm -C web dev` → the map fills the bottom band with light Positron tiles, not a grey box; the attribution line reads "© OpenStreetMap contributors © CARTO" with working links; the browser console shows no errors.

## Out of scope (do not do)

- Markers, bubble radius, colour, tooltips, selection styling — 07-02/07-03.
- Map events (`moveend`, map click), the legend, zoom-control placement, `scrollWheelZoom` — 07-04/07-05/07-10/07-11.
- Layout: the 45 vh band and the 340 px left panel belong to 05-18/07-06; do not add a height here.
- A tile-provider switch, a second tile layer, clustering, `fitBounds`, `maxBounds`, geodesic anything, or any new dependency beyond the three pinned; no change to `App.tsx`, `vite.config.ts` or the Tailwind config.

## Definition of done

- [ ] All eight cases exist in `MapView.test.tsx`, named as written, and pass.
- [ ] Red run observed and quoted; the tile URL and the attribution are asserted as exact/containing strings, not approximately.
- [ ] `leaflet`, `react-leaflet`, `@types/leaflet` are pinned without `^`/`~` and the lockfile is committed.
- [ ] Lane suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the five files above.
- [ ] Frontmatter `status: done`; `## Result` filled; exactly one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
07-01: render the leaflet map with carto positron tiles

- points: 2
- red: test_tile_layer_uses_carto_positron failed (Failed to resolve import "./MapView")
- add MapView with one MapContainer, CARTO Positron tiles and the OSM+CARTO attribution
- pin leaflet 1.9.4, react-leaflet 4.2.1 and @types/leaflet (already chosen in plan section 4), import leaflet.css
- add matchMedia and getBoundingClientRect stubs to the shared test setup
- suite: N passed
```

Files in the commit: exactly the five listed above plus this task file.

## Traps and review notes

- A second `MapContainer` (or a `MapContainer` per building) creates a second Leaflet instance; case 1 is the guard, and reviewers grep for `MapContainer` in `web/src` — it must appear in this file only.
- Do not assert `map.remove()` on unmount in the mocked tests: `react-leaflet` owns disposal and the mock would only assert itself. The real check is "one map, one container".
- `h-full` under a parent without a height is the classic grey box (plan Traps). The layout task gives the band its 45 vh; this task must not hard-code a height instead.
- Attribution is a licence term: dropping `CARTO` or shortening the string fails case 4 and breaks the tile terms.
- `react-leaflet@5` needs React 19 (the plan pins React 18): if `pnpm add` resolves 5.x, pin `4.2.1` explicitly.
- Reviewers: grep `web/src` for `basemaps.cartocdn.com` (one hit), `13.7563` (one hit) and `leaflet.css` (one hit).

## Plan samples are untrusted

The plan's props block and its six TDD items are illustrative. Verify prop names (`center`, `zoom`, `className`, `url`, `attribution`) and the tile URL against the installed `react-leaflet@4.2.1` and `leaflet@1.9.4` types before writing assertions (`CLAUDE.md` §6). If the installed version disagrees, follow the library and record it under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
