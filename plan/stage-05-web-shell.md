# Stage 05 — Web shell: data layer, state contract, layout skeleton

**Goal:** a running SPA that loads `dataset.json`, exposes the state contract every later component
depends on, and renders the final layout with placeholder panels. Can start **in parallel with
Stages 01–04** because it only needs the sample dataset committed here.

**Depends on:** Stage 00 (repo). **Est:** 50 min. **Agent:** B. **Files:**

```text
web/package.json, vite.config.ts, tsconfig.json, tailwind.config.ts, postcss.config.js
web/index.html, src/main.tsx, src/index.css
web/src/types.ts
web/src/api/loadDataset.ts
web/src/lib/{filter,group,viewport,format}.ts  (+ .test.ts)
web/src/state/{useDataset,useFilters,useViewport,useSelection}.ts
web/src/App.tsx
web/src/test/{setup.ts,fixtures.ts}
web/src/test/fixtures/dataset.sample.json
web/src/components/{StatusBar,EmptyState}.tsx   (+ tests)
```

---

## Task 5.1 — Scaffold

1. `pnpm create vite web --template react-ts` inside the repo, then add:
   `tailwindcss@3.4 postcss autoprefixer`, `vitest @testing-library/react @testing-library/user-event jsdom`.
2. `package.json` scripts: `dev`, `build`, `preview`, `test` (`vitest run`), `test:watch`, `e2e` (added
   in Stage 07), `typecheck` (`tsc --noEmit`).
3. `vite.config.ts`: `test: { environment: "jsdom", setupFiles: "./src/test/setup.ts", globals: true }`;
   `server.proxy` not needed (data is a static file).
4. Tailwind content globs `./index.html`, `./src/**/*.{ts,tsx}`; extend theme with the tokens from
   `02-architecture.md` §5.3 (`colors.surface`, `colors.ink`, `colors.accent`, `colors.muted`).
5. Delete the Vite demo (logo, counter, App.css) — the repo must not contain demo noise.
6. `web/src/test/setup.ts`: `@testing-library/jest-dom`, and a `ResizeObserver` stub (Leaflet/jsdom).

**Verify:** `pnpm -C web build` succeeds; `pnpm -C web test` runs 0 tests and exits 0.
Commit: `stage-05: vite react tailwind scaffold`.

---

## Task 5.2 — Types, dataset guard, sample data

**`types.ts`** mirrors `02-architecture.md` §3.3 exactly: `Dataset`, `Building`, `Unit`, `Stats`,
`SearchBlock`. Add a compile-time check that the sample JSON satisfies `Dataset`
(`satisfies Dataset` on the imported fixture, or a `type Test = Expect<Equal<…>>` in a test).

**`src/test/fixtures/dataset.sample.json`** (committed): 6 units across 3 buildings —
`p00348 The Bangkok Sathorn` (3 units, coords, Sa Thon/Bangkok), `slug:premier-thonglor` (2 units,
coords, Watthana/Bangkok), `slug:no-location` (1 unit, `lat: null`, Khlong Toei/Bangkok). Prices
between 25 000 and 50 000, sizes 45–150, bedrooms 0–3, one unit with `floor: null`.

**`loadDataset.ts`**

```ts
export async function loadDataset(url = "/data/dataset.json", fetchImpl = fetch): Promise<Dataset>
export class DatasetError extends Error { constructor(msg: string, readonly detail?: string) }
```

**TDD order** (`loadDataset.test.ts`, `vi.stubGlobal("fetch", …)`):

1. Valid payload → resolves with a `Dataset`; `schemaVersion === 1`.
2. Unsupported `schemaVersion` (2) → rejects with `DatasetError` whose message contains
   "schemaVersion".
3. Non-OK HTTP (500) → `DatasetError` with the status.
4. Malformed JSON → `DatasetError`, not a raw `SyntaxError`.
5. Missing `buildings` array → `DatasetError` naming the field.
6. `abortSignal` support: a pre-aborted signal rejects with an `AbortError`-like error (used by
   `useDataset` cleanup).

Commit: `stage-05: typed dataset loader`.

---

## Task 5.3 — Pure view-model functions (the heart, write these first)

These four modules contain **all** business logic; components stay dumb. Every rule F1–F6 from
`02-architecture.md` §5.2 is a test here.

```ts
// lib/filter.ts
export interface Filters { sizeMin: number | null; sizeMax: number | null;
                           beds: "any" | "studio" | 1 | 2 | 3 | "4+";
                           district: string | "all"; building: string }
export function filterUnits(units: Unit[], f: Filters): Unit[]
export function filterBuilding(b: Building, f: Filters): Building | null   // null when 0 matches

// lib/group.ts
export function groupByBuilding(ds: Dataset, units: Unit[]): Building[]    // recomputed ranges
export function priceRange(b: Building): string                           // "45–50k"
export function countLabel(n: number): string                             // "1 unit" | "3 units"

// lib/viewport.ts
export interface BBox { south: number; west: number; north: number; east: number }
export function inBBox(lat: number | null, lng: number | null, b: BBox): boolean
export function splitByLocation(buildings: Building[], b: BBox): { inView: Building[]; noLocation: Building[] }

// lib/format.ts
export function thb(n: number): string        // "฿45,000"
export function sqm(n: number): string        // "65 m²"
export function beds(n: number): string       // "Studio" | "1 bed" | "2 beds"
```

**TDD order** (one `.test.ts` per module, table-driven):

| Module | Cases |
|---|---|
| `filter` | inclusive bounds; `sizeMin > sizeMax` → empty result AND a console-free behaviour (no throw); `null` = unbounded; `beds: "studio"` matches only 0; `beds: 2` matches exactly 2; `"4+"` matches 4,5,6; district exact match; building substring case-insensitive and trimmed; building with 0 matching units is dropped |
| `group` | recalculated `priceMin/Max`, `sizeMin/Max`, `unitCount` from the filtered units, not from the dataset's precomputed values; empty input → `[]`; order preserved from dataset order (already sorted in Stage 03) |
| `viewport` | inside/outside each edge; `null` coords are never "in view" and always land in `noLocation`; dateline not required (Thailand only, comment it); `inView` keeps dataset order |
| `format` | `thb(45000) === "฿45,000"`; `thb(0)`; `sqm(65.5)`; `beds(0)==="Studio"`; `priceRange` `45–50k`, `45k`, `1.2M` |

Commit: `stage-05: pure filter/group/viewport/format modules`.

---

## Task 5.4 — State contract (hooks) and App composition

```ts
useDataset(url?)  → { status: "loading"|"ready"|"error"; data?: Dataset; error?: string; reload(): void }
useFilters()      → { filters: Filters; setFilter<K extends keyof Filters>(k: K, v: Filters[K]): void; reset(): void }
useViewport()     → { bbox: BBox | null; setBBox(b: BBox | null): void }   // null = "whole city"
useSelection(ds)  → { selectedId: string | null; select(id: string | null): void; selected: Building | null }
```

Composition rules (single source of truth in `App.tsx`):

```text
units      = filterUnits(allUnits, filters)
buildings  = groupByBuilding(dataset, units)          // ranges recomputed
{inView, noLocation} = splitByLocation(buildings, bbox ?? WORLD)
selected   = buildings.find(id === selectedId) ?? null
```

**TDD order** (`useFilters.test.ts`, `useSelection.test.ts`, `App.test.tsx`):

1. `useFilters` initial state = no filters; `setFilter` updates only that key; `reset` restores
   initial; setting the same value does not change the object identity (memo friendliness).
2. `useSelection` clears the selection when the selected building disappears from the filtered set
   (important: filters change → panel must not show a ghost).
3. `App.test.tsx` with the sample dataset and a stubbed fetch:
   - renders the header stats (`6 units`, `3 buildings`);
   - renders one row per building (the placeholders in this stage may be a simple `<ul>`);
   - `bbox` set to a box around Bangkok → buildings with coords listed, `no-location` excluded;
   - error state renders `EmptyState` with a retry button that re-calls `loadDataset`.

Commit: `stage-05: state hooks and app composition`.

---

## Task 5.5 — Layout skeleton

Render the final layout structure with real chrome and placeholder content (filters and the map are
filled in Stages 06–07):

```text
<App>
  <Header>            logo text, stats line, StatusBar (freshness / partial banner)
  <FilterBarSkeleton> disabled inputs, aria-hidden, 64 px
  <main class="grid grid-cols-[340px_1fr] gap-4">
     <aside> BuildingPanelSkeleton  (340 px, sticky, own scroll)
     <section> ResultsListSkeleton (own scroll)
  </main>
  <section class="h-[45vh]"> MapSkeleton (grey block, "map loads in stage 07")
</App>
```

Requirements:

- Nothing scrolls the whole page: `h-screen flex flex-col`, the two panels scroll independently.
- `StatusBar` is real logic (not a placeholder): shows `generatedAt` via `Intl.DateTimeFormat` in the
  browser locale, a warning icon + text when `stats.partial`, and a `title` tooltip with
  `stats.pagesScanned` / `stats.buildingsWithCoords`.
- `EmptyState` is real and reused by Stages 06/07: props `{ title, hint, action? }`.
- Semantic landmarks: `<header>`, `<main>`, `<aside>`, `<section aria-label="Results">`.

**Tests** (`StatusBar.test.tsx`, `App.test.tsx`): freshness string matches a frozen date;
`partial: true` renders the banner with `role="status"`; landmarks exist; no horizontal scroll
class regressions (`overflow-x-hidden` on the shell).

Commit: `stage-05: layout skeleton and status bar`.

---

## Acceptance criteria

- [ ] `pnpm -C web test` green; ≥ 25 tests
- [ ] `pnpm -C web build` produces `dist/` < 500 kB gzip total (Leaflet comes in Stage 07; measure then)
- [ ] `pnpm -C web typecheck` clean
- [ ] no `any` in `src/lib/**` or `src/state/**` (`grep -rn ": any" web/src`)
- [ ] `App` renders from `dataset.sample.json` with a stubbed fetch, no network
- [ ] `make check` green from the repo root

## Result

| Task | Tests | Status |
|---|---|---|
| 5.1 | build + typecheck | |
| 5.2 | `loadDataset.test.ts` (6) | |
| 5.3 | `filter/group/viewport/format.test.ts` (≈ 28) | |
| 5.4 | `useFilters/useSelection/App.test.tsx` (7) | |
| 5.5 | `StatusBar.test.tsx` (3) | |

## Traps

- Do not put business logic in components; if a test needs a DOM to check a filter, the filter
  belongs in `lib/`.
- `useSelection` must drop a stale selection when filters exclude that building — otherwise the left
  panel shows a building not in the list.
- Sample dataset **must** contain a building with `lat: null`; otherwise the "no location" path is
  untested until the real data surprises you.
- Do not import `leaflet` yet: jsdom + Leaflet requires the setup stub from Stage 07.
- Keep the layout scroll model (`h-screen`, internal scrolls) from day one; retrofitting it after the
  map lands is painful.
