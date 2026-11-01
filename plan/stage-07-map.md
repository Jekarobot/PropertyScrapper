# Stage 07 — Map, bubbles and viewport synchronisation

**Goal:** the bottom map with one bubble per building, selection wired to the left panel, and
`moveend` filtering the right list. Ends with a Playwright test that drags the real map.

**Depends on:** Stage 05 (contract); can run in parallel with Stage 06. **Est:** 60 min. **Agent:** B.

**Files:**

```text
web/src/components/MapView.tsx        (+ MapView.test.tsx)
web/src/components/BuildingPanel.tsx  (+ BuildingPanel.test.tsx)
web/src/components/MapLegend.tsx      (tiny, optional)
web/src/App.tsx                       (wire bbox + selection)
web/src/test/setup.ts                 (Leaflet/jsdom stubs)
web/playwright.config.ts
web/e2e/app.spec.ts
```

---

## Task 7.1 — `MapView` skeleton with CARTO tiles

Dependencies: `leaflet@1.9.4`, `react-leaflet@4.2.1`, `@types/leaflet`.
Import `leaflet/dist/leaflet.css` and set the container height with Tailwind (`h-full`).

```ts
interface MapViewProps {
  buildings: Building[];                 // already filtered, may include null coords (skipped)
  selectedId: string | null;
  onSelect: (id: string | null) => void;
  onViewportChange: (b: BBox) => void;
  initialCenter?: [number, number];      // default [13.7563, 100.5018], zoom 12
}
```

**TDD order** (`MapView.test.tsx`) — `react-leaflet` is **mocked** (`vi.mock`) so the test asserts
props, not Leaflet internals:

1. Renders `MapContainer` once; child `TileLayer` receives
   `url="https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png"` and attribution containing
   both `OpenStreetMap` and `CARTO` (licence requirement — assert the text).
2. One `CircleMarker` per building with non-null coords; buildings with null coords produce none.
3. Marker `center` equals `[lat, lng]` for each building; radius follows F7
   (`8 + 4*sqrt(count)` clamped 8–22); selected marker has the accent colour class/`pathOptions`.
4. `CircleMarker` click → `onSelect(id)`; map click (no marker) → `onSelect(null)`.
5. A `Tooltip` per marker shows `name`, `countLabel(unitCount)`, `priceRange`.
6. Empty `buildings` array still renders the map (no crash).

`setup.ts` additions: `ResizeObserver` no-op class, `window.matchMedia` stub, and
`HTMLElement.prototype.getBoundingClientRect` returning 800×400 (Leaflet needs non-zero sizes).

Commit: `stage-07: leaflet map with building bubbles`.

---

## Task 7.2 — Viewport tracking (`moveend` only)

**TDD order**

1. `useViewport` (Stage 05) is driven by a `<MapEvents onViewportChange>` child that registers
   `map.on("moveend", …)`; on unmount it unsubscribes (assert with the mocked map: `off` called).
2. `moveend` fires `onViewportChange` with `{south, west, north, east}` from `map.getBounds()`.
3. `move`/`zoom`/`zoomstart` do **not** fire it (F8).
4. Rapid double `moveend` within 200 ms → `onViewportChange` called once with the last bounds
   (throttle inside `MapEvents`, not in `App`).
5. `App` integration: with a bbox around Bangkok, `ResultsList` shows only buildings inside it, and
   `MapView` receives the same filtered list (one source of truth, no second filter in the map).
6. A building exactly on the bbox edge is included (inclusive test in `lib/viewport` — already
   covered in Stage 05; assert it end-to-end here once).

Commit: `stage-07: viewport sync on moveend`.

---

## Task 7.3 — Selection and the left panel

`BuildingPanel` props: `{ building: Building | null; onClose(): void }`.

**TDD order** (`BuildingPanel.test.tsx`):

1. `null` → an instructional placeholder ("Select a building on the map or in the list"), no empty box.
2. With a building → name (h2), `district · province`, `countLabel`, `priceRange`, size range.
3. Units list identical in markup/behaviour to the expanded `BuildingRow` units (reuse the same
   `UnitRow` component — DRY; extract it from Stage 06 into `components/UnitRow.tsx`).
4. A `Close` button calls `onClose`; accessible name "Clear selection".
5. `Open in maps ↗` link → `https://www.google.com/maps?q=<lat>,<lng>` only when coords exist,
   `target="_blank"` + `rel="noopener noreferrer"`.
6. For a building with null coords → the maps link is replaced by "No location" text.
7. Selection synchronisation (in `App.test.tsx`): clicking a marker selects it, the panel renders, the
   marker gets the selected style, the row gets `aria-current`; clicking the panel's close clears
   both; changing filters so the selected building disappears clears the selection (already from
   Stage 05 — assert the panel becomes the placeholder).

Commit: `stage-07: building panel and selection sync`.

---

## Task 7.4 — No-location bucket and map affordances

1. `MapLegend` (bottom-left overlay): count of visible buildings, a dot size legend (1 / 5 / 20 units)
   and the tile attribution is handled by Leaflet itself.
2. A collapsible `No location (N)` row under the results list listing buildings with `lat: null`
   (clicking selects them; the map does not move).
3. Zoom controls: Leaflet default `zoomControl` enabled, positioned `bottomright` so it does not
   collide with the left panel.
4. `scrollWheelZoom` **disabled** (the map is full-width at the bottom; page scroll must win),
   `doubleClickZoom` enabled; a hint text "Use ⌘/Ctrl + scroll to zoom" in the legend.
5. Mobile (`< 768px`): a "Map" toggle button switches between the list and the map; the filter bar
   collapses to a single "Filters (N)" button opening a sheet.

Commit: `stage-07: no-location bucket and map affordances`.

---

## Task 7.5 — Playwright: the real browser check

**Why here:** Leaflet drag cannot be tested in jsdom; this is the only proof that `moveend` →
list update actually works.

`playwright.config.ts`: `webServer: { command: "pnpm preview --port 4173", url: "http://127.0.0.1:4173" }`,
`use: { viewport: { width: 1440, height: 900 } }`, `testDir: "e2e"`, `retries: 0` locally.
Before the test, copy `web/src/test/fixtures/dataset.sample.json` to `web/public/data/dataset.json`
(a `globalSetup` script that does `mkdir -p public/data && cp …`, so e2e never depends on the real run).

**`e2e/app.spec.ts` scenarios**

1. **Load:** header shows `6 units` and `3 buildings`; 3 rows; 2 bubbles (the null-coord building is
   not a bubble).
2. **Scroll isolation:** mouse-wheel over the map doesn't scroll the page; wheel over the list
   scrolls the list only.
3. **Drag → list update:** record row names, drag the map by (−400, 0), assert the row set changed and
   the URL/hash did not change.
4. **Bubble → panel:** click a bubble → left panel shows that building's name and its units; the
   selected row is highlighted.
5. **Filter → bubbles:** set `beds=2` → bubble count decreases accordingly; clear filters restores.
6. **Link opens a new tab:** `page.expectPopup` on a unit link, assert the popup URL starts with
   `https://connex.in.th/en/detail/`.
7. **No console errors** during the whole flow (`page.on("console")` collector, fail on `error`).

Commit: `stage-07: playwright end-to-end map tests`.

---

## Acceptance criteria

- [ ] `pnpm -C web test` green (≥ 85 tests)
- [ ] `pnpm -C web e2e` green, `e2e/app.spec.ts` has 7 tests
- [ ] exactly one `MapContainer` in the tree; no Leaflet instance leaks on unmount (assert `map.remove()` called in the mocked test)
- [ ] tile URL and attribution string asserted in a test (licence compliance)
- [ ] `pnpm -C web build` → total gzip < 600 kB (Leaflet adds ~45 kB)
- [ ] dragging the map updates the list within 200 ms in the Playwright run

## Result

| Task | Tests | Status |
|---|---|---|
| 7.1 | `MapView.test.tsx` (6) | |
| 7.2 | `MapEvents`/`App.test.tsx` (6) | |
| 7.3 | `BuildingPanel.test.tsx` + `App.test.tsx` (7) | |
| 7.4 | `MapLegend`/`NoLocation` (4) | |
| 7.5 | `e2e/app.spec.ts` (7) | |

## Traps

- Do not store `map` in React state; use a `ref` inside `MapEvents` (`useMap()`).
- `moveend` fires on programmatic `fitBounds` too — guard against feedback loops by comparing the
  new bbox with the current one (stringify with 6 decimals) before calling `onViewportChange`.
- Leaflet needs a sized container; a `h-full` parent without an explicit height yields a grey box.
- `scrollWheelZoom` must be off, otherwise the bottom map "steals" page scrolling and the app feels
  broken on a laptop trackpad.
- Playwright + Vite preview needs the dataset present **before** the server starts; a `globalSetup`
  copy is the simplest reliable ordering.
- If `react-leaflet@5` gets installed, React 19 is required — pin `react-leaflet@4.2.1` and
  `react@18`.
