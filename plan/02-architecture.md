# 02 — Architecture, data model, stack and UI contract

Read after [`01-discovery.md`](01-discovery.md). This document is the reference every stage doc
points back to; if a stage conflicts with it, this document wins and must be updated first.

---

## 1. Shape of the system

```
                 ┌──────────────────────────── pipeline (Python) ────────────────────────────┐
 connex.in.th    │                                                                            │
 search_en  ──►  │  crawl ──► data/raw/<runId>/pages/page-0001.json …  (+ manifest.json)      │
 (16 cards,      │                  │                                                         │
  3 states)      │                  ▼                                                         │
                 │  build  ──► group by building ──► data/runs/<date>/dataset.json            │
 detail page ──► │  enrich ──► coordinates per building ──► data/cache/buildings.json         │
 (1 per bldg)    │                  │                                                         │
                 └──────────────────┼─────────────────────────────────────────────────────────┘
                                    ▼
                     web/public/data/dataset.json   (single file, ~0.5–1 MB)
                                    │
                                    ▼
                     Vite + React + Leaflet SPA (static, no backend)
                     filters ─► unit set ─► buildings ─► viewport ─► list + map bubbles
```

Two processes only: a **scheduled pipeline** and a **static site**. No API server, no database, no
auth. The dataset file is the contract between them.

---

## 2. Repository layout (created in Stage 00)

```
connex-watch/                       (= /home/jeka/DeepseekBench)
├── plan/                           this folder
├── Makefile                        test / check / crawl / build / publish / nightly
├── README.md                       what it is, how to run, troubleshooting
├── .gitignore
├── data/                           gitignored except data/.gitkeep
│   ├── raw/<runId>/
│   │   ├── manifest.json
│   │   └── pages/page-0001.json …
│   ├── cache/buildings.json        coordinates cache (long-lived)
│   └── runs/<YYYY-MM-DD>/dataset.json
├── pipeline/
│   ├── pyproject.toml
│   ├── presets.toml                named searches (dev, daily, …)
│   ├── src/connex_watch/
│   │   ├── __init__.py
│   │   ├── models.py               pydantic models + enums
│   │   ├── urls.py                 SearchSpec ⇄ URL
│   │   ├── search_parser.py        HTML → PageResult
│   │   ├── detail_parser.py        HTML → DetailInfo
│   │   ├── fetch.py                ThrottledFetcher (httpx, retries)
│   │   ├── crawl.py                page walk + stop policy + checkpoints
│   │   ├── store.py                raw run dir read/write, resume
│   │   ├── build_dataset.py        group/clean/dedupe → Dataset
│   │   ├── coord_cache.py          building → coords, persistent
│   │   ├── enrich.py               fill coords, merge by project_code
│   │   └── cli.py                  crawl | build | check
│   └── tests/
│       ├── conftest.py             fixture loaders
│       ├── fixtures/*.html         copied from plan/assets/fixtures
│       └── test_*.py
├── web/
│   ├── package.json                pnpm
│   ├── vite.config.ts
│   ├── tsconfig.json
│   ├── tailwind.config.ts
│   ├── index.html
│   ├── public/data/dataset.json    published by `make publish`
│   ├── playwright.config.ts
│   ├── e2e/app.spec.ts
│   └── src/
│       ├── main.tsx
│       ├── App.tsx
│       ├── types.ts
│       ├── api/loadDataset.ts
│       ├── lib/filter.ts           pure: filters → units
│       ├── lib/group.ts            pure: units → buildings
│       ├── lib/viewport.ts         pure: bbox test
│       ├── lib/format.ts           pure: ฿45,000, "65 m²"
│       ├── state/useDataset.ts
│       ├── state/useFilters.ts
│       ├── state/useViewport.ts
│       ├── state/useSelection.ts
│       ├── components/{FilterBar,ResultsList,BuildingPanel,MapView,StatusBar,EmptyState}.tsx
│       └── test/{fixtures.ts,setup.ts}
└── scripts/
    ├── nightly.sh                  flock + crawl + build + check + publish + retention
    └── connex-watch.service/.timer  systemd units (documented, not installed)
```

`data/` is gitignored because it is regenerated; `web/public/data/dataset.json` is produced by
`make publish` and gitignored too. Tests use `web/src/test/fixtures/dataset.sample.json` (committed,
3 buildings, 6 units).

---

## 3. Data model

### 3.1 Domain objects

| Object | Identity | Lives in |
|---|---|---|
| `UnitCard` | `CX-<digits>` | one search card (raw layer) |
| `Unit` | `CX-<digits>` | dataset, cleaned and filtered to `available` |
| `Building` | `project_code` when known, else `slug(building_name)` | dataset, groups units |
| `SearchSpec` | the query params | manifest + dataset `search` block |

**Unit state enum:** `available` \| `check_availability` \| `not_available`. Only `available` is
ever published. The other two are counted in `stats.excluded` so a sudden spike is visible.

### 3.2 Raw layer — `data/raw/<runId>/pages/page-0001.json`

One file per page, written **after** the page is parsed (atomic: write `.tmp`, `os.replace`).
Existing file ⇒ page already done ⇒ `--resume` skips it.

```json
{
  "schemaVersion": 1,
  "pageNo": 1,
  "totalPages": 546,
  "totalUnits": 8734,
  "fetchedAt": "2026-11-01T21:40:12Z",
  "url": "https://connex.in.th/search_en?project_code=…&page_no=1",
  "stateCounts": { "available": 16, "check_availability": 0, "not_available": 0 },
  "cards": [
    {
      "id": "CX-84121",
      "url": "https://connex.in.th/en/detail/for-rent-condo-the-bangkok-sathorn-…-cx-84121",
      "state": "available",
      "priceThb": 45000,
      "bedrooms": 1,
      "bathrooms": 1,
      "floor": 27,
      "sizeSqm": 65,
      "buildingName": "The Bangkok Sathorn",
      "district": "Sa Thon",
      "province": "Bangkok",
      "updatedOn": "2026-10-29",
      "imageUrl": "https://connex.in.th/uploads/listing/CX-84121/webp/mini_w500/01_h20zs.webp",
      "position": 1
    }
  ]
}
```

`manifest.json` holds the run: `runId`, `startedAt`, `finishedAt`, `spec`, `pagesFetched`,
`stopReason` (`last_page` \| `empty_streak` \| `max_pages` \| `aborted`), `requestCount`,
`errorCount`, `notes[]`.

### 3.3 Published dataset — `web/public/data/dataset.json`

Single file, atomic publish, self-describing. ~1100 units ≈ 1000 buildings ⇒ 0.5–1 MB.

```json
{
  "schemaVersion": 1,
  "generatedAt": "2026-11-01T21:45:00Z",
  "source": { "site": "connex.in.th", "kind": "rent-condo" },
  "search": {
    "project_code": "", "listingtype": 2, "propertytype": 1, "bedroom": null,
    "minsize": 45, "maxsize": 150, "minprice": 25000, "maxprice": 50000,
    "special": null, "sort": ""
  },
  "stats": {
    "units": 1132,
    "buildings": 987,
    "buildingsWithCoords": 940,
    "pagesScanned": 73,
    "duplicatesDropped": 2,
    "excluded": { "check_availability": 640, "not_available": 5900 },
    "partial": false
  },
  "buildings": [
    {
      "id": "p00348",
      "name": "The Bangkok Sathorn",
      "district": "Sa Thon",
      "province": "Bangkok",
      "lat": 13.7187616,
      "lng": 100.5202356,
      "coordSource": "detail:CX-84121",
      "unitCount": 3,
      "priceMin": 45000,
      "priceMax": 50000,
      "sizeMin": 58,
      "sizeMax": 65,
      "units": [
        {
          "id": "CX-84121",
          "url": "https://connex.in.th/en/detail/…-cx-84121",
          "priceThb": 45000,
          "sizeSqm": 65,
          "bedrooms": 1,
          "bathrooms": 1,
          "floor": 27,
          "updatedOn": "2026-10-29"
        }
      ]
    }
  ]
}
```

Rules baked into the schema:

- `buildings[]` sorted by `name` (case-insensitive), then `id`; `units[]` sorted by `priceThb`, then
  `id`. Deterministic output ⇒ meaningful git diffs and stable snapshots.
- `lat`/`lng` may be `null` (whole building). A building with `null` coords is still listed, just
  not mapped.
- `district` may be `null`.
- `stats.partial = true` when the crawl was aborted or a page failed; the UI shows a warning banner.
- No HTML, no description text, no images beyond nothing (YAGNI — the map + link is the product).

### 3.4 Why one file, not `units.json` + `buildings.json`

Atomic publish (rename), one fetch, no chance of a UI showing buildings from run A and units from
run B. 1 MB is well under what a laptop browser minds, and every filter is in-memory anyway.
Split only if the file ever exceeds ~10 MB.

---

## 4. Stack decisions

| Area | Choice | Why | Rejected |
|---|---|---|---|
| HTTP | `httpx` (sync client) | one dependency, timeouts, redirects, HTTP/2 to Cloudflare | `requests` (no HTTP/2, less pleasant timeouts), `curl` subprocess |
| HTML | `beautifulsoup4` + `lxml` | forgiving on malformed legacy markup, `select` CSS API, easy to read in tests | `selectolax` (faster, but 72 pages/run makes speed irrelevant), regex-only (brittle) |
| Models | `pydantic` v2 | free validation + `.model_dump_json()`; doubles as schema test | dataclasses (no validation), attrs |
| CLI | `argparse` (stdlib) | 3 subcommands; zero deps | `typer`, `click` |
| Logging | `logging` + `rich` optional | structured lines, `--verbose`; skip `rich` if it complicates | — |
| Test runner | `pytest` (+`pytest-cov`) | standard | unittest |
| Lint/format | `ruff` (lint + format) | one tool, fast | black+flake8+isort |
| Frontend build | Vite 5 | instant dev server, static output | Next.js (SSR unused, heavier), CRA (dead) |
| UI | React 18 + TypeScript 5 | ecosystem for Leaflet, typed props | Svelte (fine, but fewer map examples), Vue |
| Styles | Tailwind 3.4 + a few CSS vars | minimalistic, no component library | MUI/Bootstrap (heavy, opinionated), CSS-in-JS |
| Map | `leaflet` 1.9 + `react-leaflet` 4.2 | the standard, no API key, no terms to accept | Google Maps (key + billing), MapLibre GL (heavier, vector tiles need a key), Mapbox (key) |
| Tiles | CARTO "Positron" (`basemaps.cartocdn.com/light_all`) | light, minimal, fits the design, free for low volume, attribution to OSM+CARTO | raw OSM tiles (busy look, stricter usage policy) — keep as one-line env switch |
| Unit tests | `vitest` + `@testing-library/react` + `jsdom` | same config as Vite | jest (config tax) |
| E2E | `playwright` | real browser needed to test Leaflet drag | cypress (heavier) |
| Package manager | `pnpm` | installed (12.16.1), fast, strict | npm |

Node 22, Python 3.12 are installed on the target machine. Pin exact versions in `package.json` and
`pyproject.toml`; commit `pnpm-lock.yaml`.

**Leaflet in jsdom:** Leaflet needs layout; component tests mock `react-leaflet` and assert props
(marker positions, bounds). Real drag/move behaviour is covered by Playwright only. Do not fight
jsdom.

---

## 5. UI specification (what "minimal, clean, modern" means here)

### 5.1 Layout

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ CONNEX WATCH · 1 132 units · 987 buildings · updated 01.11 21:45   [data ⓘ]  │  header 56px
├──────────────────────────────────────────────────────────────────────────────┤
│ size [45]–[150] m² │ beds [Any ▾] │ district [All ▾] │ building [search…] ⟲  │  filter bar 64px
├───────────────────────────┬──────────────────────────────────────────────────┤
│ LEFT 340px                │ RIGHT (flex)                                     │
│ ┌───────────────────────┐ │ ┌──────────────────────────────────────────────┐ │
│ │ The Bangkok Sathorn   │ │ │ ▸ KnightsBridge Prime Sathorn  2 units  45–50k│ │
│ │ Sa Thon, Bangkok      │ │ │ ▸ Premier Thonglor             1 unit   32k   │ │
│ │ 3 units · 45–50k      │ │ │ …                                            │ │
│ │ [open on maps ↗]      │ │ │                                              │ │
│ │ ─────────────────     │ │ │ (scrollable, viewport-filtered)              │ │
│ │ CX-84121  65 m² 1BR   │ │ │                                              │ │
│ │ ฿45,000  [view ↗]     │ │ │                                              │ │
│ │ …                     │ │ └──────────────────────────────────────────────┘ │
│ └───────────────────────┘ │                                                  │
├───────────────────────────┴──────────────────────────────────────────────────┤
│ MAP (full width, 45vh, CARTO Positron, one bubble per building)               │
└──────────────────────────────────────────────────────────────────────────────┘
```

Decisions:

- **Bottom map is explicit in the request** — keep it, full width, 45vh, sticky to the bottom of
  the viewport.
- **Left = units of the selected building**; **right = buildings visible in the current map
  viewport** (after filters). Selecting a building (bubble or row) fills the left panel and
  highlights the bubble. This satisfies both "left shows units that building has" and "when the map
  moves, update the list on the right".
- Rows are **buildings**, not units, because "group by condo" is the product's point. Each row shows
  name, district, unit count, price range. Expanding a row shows its units inline (same component as
  the left panel list) — one extra affordance, no extra screen.
- Empty state: "No buildings in this area — zoom out or clear filters" with a reset button.
- The unit link always opens the site in a new tab: `target="_blank" rel="noopener noreferrer"`.

### 5.2 Behaviour contract (testable statements)

| Rule | Statement |
|---|---|
| F1 | `size` filters are inclusive on both ends; empty means "no bound"; invalid input is ignored, not NaN |
| F2 | `beds` is exact (`Any`, `Studio` = 0, `1`…`4+`); `4+` matches `>= 4` |
| F3 | `district` matches the dataset `district` value exactly; `All` = no filter |
| F4 | `building` is a case-insensitive substring match on `building.name`; trimmed; debounce 200 ms |
| F5 | Filters apply before the viewport cut; a building is visible if it has ≥1 unit passing filters |
| F6 | Viewport cut uses the building's coordinates; buildings with null coords are never in the viewport list but are listed in a collapsible "No location (N)" section |
| F7 | Bubble radius encodes `unitCount` (`8 + 4·sqrt(n)` px, clamped 8–22); colour = single accent, selected = darker |
| F8 | Map `moveend` updates the right list within 200 ms; `zoomstart`/`movestart` do not |
| F9 | Selecting a building sets `selectedId`; deselecting (X or clicking empty map) clears the left panel |
| F10 | Filters/selection/viewport live in React state only (no URL sync in v1) |
| F11 | Header shows `generatedAt` in local time and a warning banner when `stats.partial` |

### 5.3 Style tokens (Tailwind)

- Background `#fafafa`, surface `#ffffff`, border `#e5e7eb`, text `#111827`, muted `#6b7280`.
- Accent `#0f766e` (teal-700); selected bubble `#0f766e`, idle `#14b8a6` at 80 % opacity.
- Type: system UI stack, 14 px base, `tabular-nums` for prices, `font-medium` for building names.
- Radius 10 px cards, 8 px bubbles, subtle shadow `shadow-sm` only on the header and the selected
  panel. No gradients, no icon library beyond 4 inline SVGs.

---

## 6. Pipeline behaviour contract

| Rule | Statement |
|---|---|
| P1 | Every network call goes through `ThrottledFetcher`; no bare `httpx` calls elsewhere |
| P2 | A page with zero cards and `pageNo < totalPages` is an **error**, not end-of-data (protects against R1) |
| P3 | A page with zero cards and `pageNo > totalPages`, or the `error/th` stub, is end-of-data |
| P4 | Stop policy: stop when `emptyStreak >= 2`, or `pageNo == totalPages`, or `--max-pages`, whichever first; `--full` ignores the streak |
| P5 | Only `state == available` cards reach the dataset; the other two are counted |
| P6 | Coordinates are fetched at most once per building per run and cached across runs; cache hits are logged |
| P7 | Grouping key = normalised name; groups sharing a `project_code` from enrichment merge into one building |
| P8 | Output ordering is deterministic (§3.3) — same input yields byte-identical JSON except `generatedAt` |
| P9 | `check` exits non-zero if: 0 buildings, 0 units, any unit without `url`, `stats.partial`, or `generatedAt` older than 48 h |
| P10 | Every run is resumable: re-running `crawl --resume` skips pages already on disk and appends nothing twice |

---

## 7. Performance targets

| Target | Budget |
|---|---|
| nightly crawl (warm cache) | < 4 min, < 80 requests |
| first cold run (coords for ~1000 buildings) | < 30 min, ~1100 requests |
| `build` (no network) | < 2 s |
| dataset load in browser | < 300 ms for 1 MB |
| filter + viewport recompute | < 30 ms (memoised, ~1000 buildings) |
| initial paint (no map tiles) | < 1 s on localhost |

---

## 8. Security / privacy / compliance

- No secrets: no Google API key is used, only coordinates are read out of the public HTML.
- Data collected is public listing data; no personal data is scraped (`process_contact.php` forms,
  phone numbers and agent names are **never** touched).
- Robots.txt allows the paths we use.
- Identify the crawler: `ConnexWatch/1.0 (+internal tool; contact: <owner email>)`.
- Throttle and backoff are non-negotiable defaults; `--delay` has a floor of `1.0 s` enforced in code.
- The site is read-only for us: no POSTs, no logins, no form submissions.

---

## 9. Explicit non-goals for v1 (backlog, do not build)

1. Cross-run history/diff ("new this week", "price dropped") — needs a runs table; the raw layer
   already keeps the data, so this is additive later.
2. Sale listings (`listingtype=1`) — the pipeline is parameterised, the UI copy says "rent".
3. Server-side filtering or a backend of any kind.
4. User accounts, saved searches, email/Slack notifications.
5. Photos/descriptions in the UI.
6. Multi-city presets and per-preset datasets.
7. i18n (the UI is English; the data has Thai names in some fields).

Each is one paragraph of future work in `README.md` under "Roadmap", nothing more.
