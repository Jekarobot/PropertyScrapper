# Tasks — the executable breakdown

This folder turns `plan/` into 129 bite-sized, TDD-first tasks. Each file is a complete brief for one
agent: what to do, what not to do, how to prove it, and how to commit it. **Read
[`../CLAUDE.md`](../CLAUDE.md) first** — it is the working agreement (trunk = `main`, one task = one
commit, test first, no code in plans, plan samples are untrusted).

Nothing here is imported by the product. These are instructions, not source code.

---

## 1. Scale — story points

| Points | Meaning | Typical tasks here |
|---|---|---|
| **0.5** | One simple change + its direct test (a formatter, a guard, one more parsed field). | `04-08`, `06-11`, `08-03` |
| **1** | A normal change with tests (a pure function, a module, a CLI flag, a component branch). | `03-02`, `05-04`, `06-06` |
| **2** | Medium-heavy (a page parser, a state machine, a feature component, a map layer). | `01-05`, `05-17`, `07-01` |
| **3** | Complex: investigate first, then implement; the task records what was learned. | `07-15`, `08-12` |

Rules of thumb: if the task needs more than one sentence to name its single outcome, or needs files
from two lanes, it is cut too big. If it needs no test, it is cut too small (fold it into a
neighbour).

Totals: **129 tasks · 132 points**. Pipeline lane (stages 00–04): 69 tasks / 67 points. Web lane
(stages 05–07): 48 tasks / 52 points. Ops/release (stage 08): 12 tasks / 13 points.

---

## 2. How to execute one task

1. Open the task file, read it end to end, then the documents under "Read before you start".
2. Set its frontmatter `status: in-progress`.
3. Write the listed tests, run them, confirm they fail for the stated reason, keep the failure line.
4. Implement the minimum, run the target tests, then the lane suite.
5. Run the task's verification commands; fill `## Result`; set `status: done`.
6. Commit **one** commit to the trunk with the task's message, including only the listed files.

A task file is also the prompt for a fresh sub-agent; the hand-off text is in `CLAUDE.md` §12.
Never run two tasks concurrently if their file lists intersect or one depends on the other.

---

## 3. Index

Legend: *Depends on* lists task ids that must be `status: done` first. `—` means nothing but the
repository baseline. Stage-level notes are in the stage table after each list.

### Stage 00 — Bootstrap (6 tasks, 4 points)

| ID | File | Task | Pts | Depends on |
|---|---|---|---|---|
| 00-01 | [task-01-repo-baseline.md](stage-00/task-01-repo-baseline.md) | Verify trunk state, add ignore rules and `data/.gitkeep` | 0.5 | — |
| 00-02 | [task-02-readme-skeleton.md](stage-00/task-02-readme-skeleton.md) | Repository README skeleton | 0.5 | 00-01 |
| 00-03 | [task-03-python-package.md](stage-00/task-03-python-package.md) | Python package, pytest and ruff configuration | 1 | 00-01 |
| 00-04 | [task-04-html-fixtures.md](stage-00/task-04-html-fixtures.md) | Copy frozen HTML fixtures and provenance README | 0.5 | 00-03 |
| 00-05 | [task-05-conftest-loaders.md](stage-00/task-05-conftest-loaders.md) | Pytest fixture loaders | 0.5 | 00-04 |
| 00-06 | [task-06-makefile.md](stage-00/task-06-makefile.md) | Makefile with `test`/`check` entry points | 1 | 00-03, 00-05 |

### Stage 01 — Search-page parser (17 tasks, 14.5 points)

| ID | File | Task | Pts | Depends on |
|---|---|---|---|---|
| 01-01 | [task-01-unit-state-enum.md](stage-01/task-01-unit-state-enum.md) | `UnitState` enum and label mapping | 0.5 | 00-03 |
| 01-02 | [task-02-card-models.md](stage-01/task-02-card-models.md) | `UnitCard`/`PageResult` models and validation | 1 | 01-01 |
| 01-03 | [task-03-search-spec-from-url.md](stage-01/task-03-search-spec-from-url.md) | `SearchSpec` defaults and `from_url` | 1 | 01-02 |
| 01-04 | [task-04-search-spec-to-url.md](stage-01/task-04-search-spec-to-url.md) | Canonical `to_url`, round-trip, unknown params | 1 | 01-03 |
| 01-05 | [task-05-card-discovery.md](stage-01/task-05-card-discovery.md) | Discover card nodes, positions and links | 2 | 01-02, 00-05 |
| 01-06 | [task-06-state-histogram.md](stage-01/task-06-state-histogram.md) | Tri-state extraction and per-page histogram | 1 | 01-05 |
| 01-07 | [task-07-missing-state-error.md](stage-01/task-07-missing-state-error.md) | Missing state block raises a located error | 0.5 | 01-06 |
| 01-08 | [task-08-error-stub-detection.md](stage-01/task-08-error-stub-detection.md) | Detect the out-of-range error stub | 0.5 | 01-05 |
| 01-09 | [task-09-price-parsing.md](stage-01/task-09-price-parsing.md) | Parse price THB from the detail line | 0.5 | 01-05 |
| 01-10 | [task-10-rooms-floor-parsing.md](stage-01/task-10-rooms-floor-parsing.md) | Bedrooms, bathrooms, floor via icon labels | 1 | 01-09 |
| 01-11 | [task-11-size-parsing.md](stage-01/task-11-size-parsing.md) | Size in sqm, decimals, missing-size error | 1 | 01-10 |
| 01-12 | [task-12-identity-image.md](stage-01/task-12-identity-image.md) | Unit id, absolute URL, image URL | 1 | 01-05 |
| 01-13 | [task-13-location-from-right.md](stage-01/task-13-location-from-right.md) | Building/district/province, parsed from the right | 1 | 01-12 |
| 01-14 | [task-14-buddhist-date.md](stage-01/task-14-buddhist-date.md) | Buddhist-era date to ISO, malformed dates | 0.5 | 01-12 |
| 01-15 | [task-15-pagination-fields.md](stage-01/task-15-pagination-fields.md) | Page number and total pages from the pager | 0.5 | 01-05 |
| 01-16 | [task-16-pagination-errors.md](stage-01/task-16-pagination-errors.md) | Missing pager error; total units derived, not SEO | 0.5 | 01-15 |
| 01-17 | [task-17-golden-snapshot.md](stage-01/task-17-golden-snapshot.md) | Golden JSON snapshot of the DEV page | 1 | 01-11, 01-14, 01-16 |

### Stage 02 — Crawler (16 tasks, 17 points)

| ID | File | Task | Pts | Depends on |
|---|---|---|---|---|
| 02-01 | [task-01-fetcher-throttle.md](stage-02/task-01-fetcher-throttle.md) | Fetcher protocol, throttle and politeness floor | 1 | 01-01 |
| 02-02 | [task-02-fetcher-retries.md](stage-02/task-02-fetcher-retries.md) | Retries, exponential backoff, `Retry-After` | 2 | 02-01 |
| 02-03 | [task-03-fetcher-refusals.md](stage-02/task-03-fetcher-refusals.md) | 403 abort, 404, UTF-8 decoding, client settings | 1 | 02-01 |
| 02-04 | [task-04-stop-policy-models.md](stage-02/task-04-stop-policy-models.md) | `StopPolicy` and `CrawlResult` models | 0.5 | 01-02 |
| 02-05 | [task-05-crawl-loop.md](stage-02/task-05-crawl-loop.md) | Page walk with fake fetcher and empty streak | 2 | 02-04, 01-04, 01-17 |
| 02-06 | [task-06-stop-reasons.md](stage-02/task-06-stop-reasons.md) | Last page, max pages, full scan stop reasons | 1 | 02-05 |
| 02-07 | [task-07-card-error-and-stub.md](stage-02/task-07-card-error-and-stub.md) | In-range zero cards is an error; stub ends the walk | 1 | 02-05 |
| 02-08 | [task-08-duplicate-counting.md](stage-02/task-08-duplicate-counting.md) | Count duplicate unit ids, do not drop here | 0.5 | 02-05 |
| 02-09 | [task-09-run-store.md](stage-02/task-09-run-store.md) | Run directory, manifest creation | 1 | 02-04 |
| 02-10 | [task-10-page-checkpoint.md](stage-02/task-10-page-checkpoint.md) | Atomic page checkpoint round-trip | 1 | 02-09 |
| 02-11 | [task-11-resume.md](stage-02/task-11-resume.md) | Resume skips completed pages | 1 | 02-10 |
| 02-12 | [task-12-manifest-finish.md](stage-02/task-12-manifest-finish.md) | Finish manifest, spec-mismatch guard | 1 | 02-09 |
| 02-13 | [task-13-presets-cli-dry-run.md](stage-02/task-13-presets-cli-dry-run.md) | Presets, `crawl` CLI, dry-run plan | 2 | 02-05, 02-09 |
| 02-14 | [task-14-cli-exit-codes.md](stage-02/task-14-cli-exit-codes.md) | CLI exit codes and operator messages | 1 | 02-13 |
| 02-15 | [task-15-cli-delay-logging.md](stage-02/task-15-cli-delay-logging.md) | Delay clamp and per-page verbose logging | 0.5 | 02-13 |
| 02-16 | [task-16-network-smoke.md](stage-02/task-16-network-smoke.md) | Opt-in live smoke test | 0.5 | 02-14 |

### Stage 03 — Dataset builder (15 tasks, 14.5 points)

| ID | File | Task | Pts | Depends on |
|---|---|---|---|---|
| 03-01 | [task-01-run-mini-fixture.md](stage-03/task-01-run-mini-fixture.md) | Hand-built `run-mini` raw fixture | 1 | 01-02 |
| 03-02 | [task-02-normalize-name.md](stage-03/task-02-normalize-name.md) | Name normalisation (NFKC, case, punctuation) | 0.5 | 01-02 |
| 03-03 | [task-03-slugify-display-name.md](stage-03/task-03-slugify-display-name.md) | Slug and display-name selection | 1 | 03-02 |
| 03-04 | [task-04-parse-location.md](stage-03/task-04-parse-location.md) | Location tuple from split parts | 0.5 | 03-02 |
| 03-05 | [task-05-available-filter.md](stage-03/task-05-available-filter.md) | Keep only available units, count exclusions | 1 | 03-01, 03-02 |
| 03-06 | [task-06-dedupe-invalid.md](stage-03/task-06-dedupe-invalid.md) | First-wins dedupe and invalid-unit accounting | 1 | 03-05 |
| 03-07 | [task-07-empty-dataset-error.md](stage-03/task-07-empty-dataset-error.md) | Refuse to build an empty dataset | 0.5 | 03-05 |
| 03-08 | [task-08-group-buildings.md](stage-03/task-08-group-buildings.md) | Group units into buildings and recompute ranges | 2 | 03-02, 03-03 |
| 03-09 | [task-09-building-ids.md](stage-03/task-09-building-ids.md) | Building ids, slug collisions | 0.5 | 03-08 |
| 03-10 | [task-10-district-ordering.md](stage-03/task-10-district-ordering.md) | District/province vote and deterministic ordering | 1 | 03-08 |
| 03-11 | [task-11-dataset-assembly.md](stage-03/task-11-dataset-assembly.md) | Assemble dataset and stats, frozen clock | 2 | 03-06, 03-10 |
| 03-12 | [task-12-validate-dataset.md](stage-03/task-12-validate-dataset.md) | Dataset invariants | 1 | 03-11 |
| 03-13 | [task-13-build-cli.md](stage-03/task-13-build-cli.md) | `build` subcommand, atomic write | 1 | 03-11 |
| 03-14 | [task-14-check-cli.md](stage-03/task-14-check-cli.md) | `check` subcommand and exit policy | 0.5 | 03-12 |
| 03-15 | [task-15-edge-cases.md](stage-03/task-15-edge-cases.md) | Grouping edge cases and degraded input | 1 | 03-11 |

### Stage 04 — Enrichment (15 tasks, 17 points)

| ID | File | Task | Pts | Depends on |
|---|---|---|---|---|
| 04-01 | [task-01-detail-coords.md](stage-04/task-01-detail-coords.md) | Coordinates from the detail page | 2 | 01-02, 00-04 |
| 04-02 | [task-02-detail-identity.md](stage-04/task-02-detail-identity.md) | Project code, breadcrumb name, unit code | 1 | 04-01 |
| 04-03 | [task-03-detail-reasons.md](stage-04/task-03-detail-reasons.md) | Missing/out-of-bounds coords and reasons | 1 | 04-01 |
| 04-04 | [task-04-detail-no-state.md](stage-04/task-04-detail-no-state.md) | Detail page never yields an availability state | 0.5 | 04-01 |
| 04-05 | [task-05-coord-cache.md](stage-04/task-05-coord-cache.md) | Coordinate cache read/write | 1 | 04-02 |
| 04-06 | [task-06-cache-negative-ttl.md](stage-04/task-06-cache-negative-ttl.md) | Negative cache with retry TTL | 1 | 04-05 |
| 04-07 | [task-07-cache-corruption.md](stage-04/task-07-cache-corruption.md) | Corrupt cache error and atomic save | 1 | 04-05 |
| 04-08 | [task-08-cache-helpers.md](stage-04/task-08-cache-helpers.md) | Cache size and pending helpers | 0.5 | 04-05 |
| 04-09 | [task-09-enrich-loop.md](stage-04/task-09-enrich-loop.md) | One detail fetch per building, cheapest unit | 2 | 04-05, 03-11, 02-01 |
| 04-10 | [task-10-enrich-caps.md](stage-04/task-10-enrich-caps.md) | Fetch cap and consecutive-failure abort | 1 | 04-09 |
| 04-11 | [task-11-enrich-report.md](stage-04/task-11-enrich-report.md) | Enrichment report and progress logging | 0.5 | 04-09 |
| 04-12 | [task-12-merge-project-code.md](stage-04/task-12-merge-project-code.md) | Merge buildings by project code | 2 | 04-09 |
| 04-13 | [task-13-merge-collisions.md](stage-04/task-13-merge-collisions.md) | Same-name different projects stay separate | 0.5 | 04-12 |
| 04-14 | [task-14-build-cli-coords.md](stage-04/task-14-build-cli-coords.md) | `build --coords/--no-coords/--dry-run` | 2 | 04-09, 03-13 |
| 04-15 | [task-15-build-cli-failures.md](stage-04/task-15-build-cli-failures.md) | Partial enrichment and corrupt-cache exit | 1 | 04-14 |

### Stage 05 — Web shell (20 tasks, 18.5 points)

| ID | File | Task | Pts | Depends on |
|---|---|---|---|---|
| 05-01 | [task-01-vite-scaffold.md](stage-05/task-01-vite-scaffold.md) | Vite + React + TS + Tailwind + Vitest scaffold | 2 | 00-06 |
| 05-02 | [task-02-test-setup.md](stage-05/task-02-test-setup.md) | Test setup, jsdom stubs, demo cleanup | 0.5 | 05-01 |
| 05-03 | [task-03-types-sample-data.md](stage-05/task-03-types-sample-data.md) | Dataset types and sample dataset | 1 | 05-01 |
| 05-04 | [task-04-load-dataset.md](stage-05/task-04-load-dataset.md) | Typed dataset loader, happy path | 1 | 05-03 |
| 05-05 | [task-05-load-dataset-errors.md](stage-05/task-05-load-dataset-errors.md) | Loader errors: schema, HTTP, JSON, shape | 1 | 05-04 |
| 05-06 | [task-06-load-dataset-abort.md](stage-05/task-06-load-dataset-abort.md) | Abort signal support in the loader | 0.5 | 05-04 |
| 05-07 | [task-07-filter-size.md](stage-05/task-07-filter-size.md) | Size bounds filter | 1 | 05-03 |
| 05-08 | [task-08-filter-beds.md](stage-05/task-08-filter-beds.md) | Bedrooms filter including studio and 4+ | 0.5 | 05-07 |
| 05-09 | [task-09-filter-district-building.md](stage-05/task-09-filter-district-building.md) | District and building-text filters | 1 | 05-08 |
| 05-10 | [task-10-group-buildings.md](stage-05/task-10-group-buildings.md) | Group filtered units, recompute ranges, labels | 1 | 05-03 |
| 05-11 | [task-11-viewport.md](stage-05/task-11-viewport.md) | Bounding-box test and no-location split | 1 | 05-03 |
| 05-12 | [task-12-format.md](stage-05/task-12-format.md) | Money, area, bed and range formatting | 1 | 05-03 |
| 05-13 | [task-13-use-dataset.md](stage-05/task-13-use-dataset.md) | Dataset loading hook with states | 1 | 05-06 |
| 05-14 | [task-14-use-filters.md](stage-05/task-14-use-filters.md) | Filters hook | 0.5 | 05-07 |
| 05-15 | [task-15-use-viewport.md](stage-05/task-15-use-viewport.md) | Viewport hook | 0.5 | 05-11 |
| 05-16 | [task-16-use-selection.md](stage-05/task-16-use-selection.md) | Selection hook with stale-selection cleanup | 0.5 | 05-10 |
| 05-17 | [task-17-app-composition.md](stage-05/task-17-app-composition.md) | App composition: one data flow | 2 | 05-13, 05-14, 05-15, 05-16 |
| 05-18 | [task-18-layout-skeleton.md](stage-05/task-18-layout-skeleton.md) | Layout skeleton and scroll model | 1 | 05-17 |
| 05-19 | [task-19-status-bar.md](stage-05/task-19-status-bar.md) | Freshness and partial-data status bar | 1 | 05-18 |
| 05-20 | [task-20-empty-state.md](stage-05/task-20-empty-state.md) | Reusable empty state component | 0.5 | 05-18 |

### Stage 06 — Filters and results list (12 tasks, 12 points)

| ID | File | Task | Pts | Depends on |
|---|---|---|---|---|
| 06-01 | [task-01-filterbar-size.md](stage-06/task-01-filterbar-size.md) | Filter bar size inputs, debounce, validation | 2 | 05-14, 05-19 |
| 06-02 | [task-02-filterbar-beds-district.md](stage-06/task-02-filterbar-beds-district.md) | Bedrooms and district controls | 1 | 06-01 |
| 06-03 | [task-03-filterbar-building.md](stage-06/task-03-filterbar-building.md) | Building search with datalist | 1 | 06-02 |
| 06-04 | [task-04-filterbar-a11y.md](stage-06/task-04-filterbar-a11y.md) | Reset, live result count, labels, tab order | 1 | 06-03 |
| 06-05 | [task-05-wire-filters.md](stage-06/task-05-wire-filters.md) | Wire filters into App | 1 | 06-04, 05-17 |
| 06-06 | [task-06-building-row.md](stage-06/task-06-building-row.md) | Building row and selection | 1 | 05-10 |
| 06-07 | [task-07-unit-row.md](stage-06/task-07-unit-row.md) | Shared unit row with safe external links | 1 | 05-12 |
| 06-08 | [task-08-building-row-expand.md](stage-06/task-08-building-row-expand.md) | Expandable row reveals units | 1 | 06-06, 06-07 |
| 06-09 | [task-09-results-list.md](stage-06/task-09-results-list.md) | Results list, loading and empty states | 1 | 06-06, 05-20 |
| 06-10 | [task-10-counts-partial.md](stage-06/task-10-counts-partial.md) | Counts header and partial-data warning | 1 | 06-09 |
| 06-11 | [task-11-list-perf.md](stage-06/task-11-list-perf.md) | Large-list render budget | 0.5 | 06-09 |
| 06-12 | [task-12-empty-error-states.md](stage-06/task-12-empty-error-states.md) | Empty and error states with actions | 0.5 | 06-09, 06-05 |

### Stage 07 — Map and selection (16 tasks, 21.5 points)

| ID | File | Task | Pts | Depends on |
|---|---|---|---|---|
| 07-01 | [task-01-map-tiles.md](stage-07/task-01-map-tiles.md) | Leaflet map with CARTO tiles and attribution | 2 | 05-01, 05-03 |
| 07-02 | [task-02-map-markers.md](stage-07/task-02-map-markers.md) | One bubble per building, radius and colour | 2 | 07-01 |
| 07-03 | [task-03-map-interactions.md](stage-07/task-03-map-interactions.md) | Marker click, map click, tooltips | 1 | 07-02 |
| 07-04 | [task-04-map-events.md](stage-07/task-04-map-events.md) | `moveend` subscription and cleanup | 1 | 07-01, 05-15 |
| 07-05 | [task-05-moveend-throttle.md](stage-07/task-05-moveend-throttle.md) | Only `moveend`, throttled, loop-guarded | 1 | 07-04 |
| 07-06 | [task-06-app-viewport-wiring.md](stage-07/task-06-app-viewport-wiring.md) | Viewport drives the results list | 1 | 07-05, 05-17 |
| 07-07 | [task-07-building-panel.md](stage-07/task-07-building-panel.md) | Left building panel with placeholder | 1 | 05-12 |
| 07-08 | [task-08-building-panel-units.md](stage-07/task-08-building-panel-units.md) | Panel content, units, close, maps link | 1 | 07-07, 06-07 |
| 07-09 | [task-09-selection-sync.md](stage-07/task-09-selection-sync.md) | One selection across map, list, panel | 1 | 07-06, 07-03, 07-08 |
| 07-10 | [task-10-legend-no-location.md](stage-07/task-10-legend-no-location.md) | Map legend and no-location bucket | 1 | 07-02 |
| 07-11 | [task-11-map-zoom-options.md](stage-07/task-11-map-zoom-options.md) | Zoom control placement, scroll-wheel policy | 0.5 | 07-01 |
| 07-12 | [task-12-mobile-toggle-sheet.md](stage-07/task-12-mobile-toggle-sheet.md) | Mobile map toggle and filter sheet | 2 | 07-09, 06-04 |
| 07-13 | [task-13-playwright-config.md](stage-07/task-13-playwright-config.md) | Playwright config and dataset bootstrap | 1 | 07-09 |
| 07-14 | [task-14-e2e-load-scroll.md](stage-07/task-14-e2e-load-scroll.md) | E2E: load counts, scroll isolation | 1 | 07-13 |
| 07-15 | [task-15-e2e-drag.md](stage-07/task-15-e2e-drag.md) | Investigate and prove drag → list update | 3 | 07-14 |
| 07-16 | [task-16-e2e-selection-popup-console.md](stage-07/task-16-e2e-selection-popup-console.md) | E2E: selection, filters, popup, console | 2 | 07-15 |

### Stage 08 — Hardening and release (12 tasks, 13 points)

| ID | File | Task | Pts | Depends on |
|---|---|---|---|---|
| 08-01 | [task-01-nightly-skeleton.md](stage-08/task-01-nightly-skeleton.md) | Nightly script skeleton and dry-run | 1 | 04-15, 02-13 |
| 08-02 | [task-02-nightly-commands.md](stage-08/task-02-nightly-commands.md) | Crawl → build → check → publish pipeline | 1 | 08-01 |
| 08-03 | [task-03-retention-rotation.md](stage-08/task-03-retention-rotation.md) | Retention and log rotation | 0.5 | 08-02 |
| 08-04 | [task-04-nightly-harness.md](stage-08/task-04-nightly-harness.md) | Stub-binary test harness for the script | 2 | 08-02 |
| 08-05 | [task-05-systemd-units.md](stage-08/task-05-systemd-units.md) | Documented systemd service and timer | 0.5 | 08-02 |
| 08-06 | [task-06-make-targets.md](stage-08/task-06-make-targets.md) | `publish`, `nightly`, `help`, final `check` | 1 | 08-02, 03-14, 04-14 |
| 08-07 | [task-07-a11y-landmarks.md](stage-08/task-07-a11y-landmarks.md) | Landmarks, headings, focus and live regions | 1 | 07-09 |
| 08-08 | [task-08-contrast-helper.md](stage-08/task-08-contrast-helper.md) | Contrast helper and accent check | 0.5 | 05-12 |
| 08-09 | [task-09-responsive-e2e.md](stage-08/task-09-responsive-e2e.md) | Mobile layout end-to-end | 1 | 07-12, 08-07 |
| 08-10 | [task-10-reduced-motion.md](stage-08/task-10-reduced-motion.md) | Reduced-motion support | 0.5 | 07-12 |
| 08-11 | [task-11-readme-runbook.md](stage-08/task-11-readme-runbook.md) | Runbook, exit codes, troubleshooting, roadmap | 1 | 08-03, 08-05 |
| 08-12 | [task-12-release-regression.md](stage-08/task-12-release-regression.md) | Full regression and release sign-off | 3 | 08-06, 08-09 |

---

## 4. Lanes and waves (sub-agent scheduling)

Two lanes run in parallel, one task at a time **inside** each lane. Pipeline and web never share a
file, so cross-lane parallelism is always safe.

| Wave | Pipeline lane (stages 00–04) | Web lane (stages 05–07) | Can start when |
|---|---|---|---|
| 0 | 00-01 → 00-02 → 00-03 → 00-04 → 00-05 → 00-06 | — | repo baseline |
| 1 | 01-01 … 01-17 (in order) | 05-01 … 05-20 (in order) | wave 0 done |
| 2 | 02-01 … 02-16 | 06-01 … 06-12 | 01 done / 05 done |
| 3 | 03-01 … 03-15 | 07-01 … 07-16 | 02 done / 06 done |
| 4 | 04-01 … 04-15 | — (review & polish) | 03 done |
| 5 | 08-01 … 08-06 (ops) | 08-07 … 08-10 (quality) | 04 done / 07 done |
| 6 | 08-11 → 08-12 (one agent, sequential) | — | wave 5 done |

Additional parallelism that is safe when you need it (disjoint files): `05-04`, `05-10`, `05-11` and
`05-12` are independent once 05-03 exists. `05-07` → `05-08` → `05-09` all edit `lib/filter.ts` and must stay
sequential; `06-06`/`06-07` can run together. Do not parallelise anything that edits `App.tsx`, `cli.py`,
`Makefile`, `README.md`, or the same test file.

After each stage: reviewer agent (read-only) re-runs the stage's commands and the stage rows of
`plan/90-test-matrix.md`, and files a findings table. Blockers must be fixed before the next stage.

---

## 5. Rules that apply to every task

- **Trunk only.** One task = one commit on `main` (the owner calls it "master"). No branches.
- **TDD.** Tests first, observed red, then green. Evidence in the commit body.
- **One thing.** Stay inside the file list; report anything else you believe is needed.
- **No code in task files.** These are instructions. Describe behaviour, name tests, cite documents.
- **Plan samples are untrusted** (`CLAUDE.md` §6). Verify against pinned versions and the repo; record
  deviations in `## Decisions`.
- **DRY / YAGNI.** One implementation per rule, no speculative features, no new dependencies.
- **No network in unit tests.** The single opt-in network test is `02-16`; live crawls need the
  owner's approval.
- **Fixtures are frozen.** Never re-capture to make a test pass; fix the parser or the assertion.
- **Bookkeeping in the same commit.** `status: done` + `## Result` land with the code.

---

## 6. Point accounting

| Stage | Tasks | Points | Cumulative points |
|---|---|---|---|
| 00 Bootstrap | 6 | 4 | 4 |
| 01 Search parser | 17 | 14.5 | 18.5 |
| 02 Crawler | 16 | 17 | 35.5 |
| 03 Dataset builder | 15 | 14.5 | 50 |
| 04 Enrichment | 15 | 17 | 67 |
| 05 Web shell | 20 | 18.5 | 85.5 |
| 06 Filters | 12 | 12 | 97.5 |
| 07 Map | 16 | 21.5 | 119 |
| 08 Release | 12 | 13 | 132 |

A 1-point task should be roughly one focused session (60–90 minutes for a competent engineer,
including tests). If your first task takes twice that, stop and report — the cut is wrong.

---

## 7. Requirement coverage — nothing from the plan is left behind

Every row of `plan/90-test-matrix.md` maps to at least one task. Use this table at stage review: a
requirement whose task is missing is a planning bug, not an implementation bug.

### Pipeline

| Req | What it proves | Task(s) |
|---|---|---|
| P-01 | Search URL round-trips losslessly (DEV + FULL) | 01-03, 01-04 |
| P-02 | Canonical parameter order, empty params preserved | 01-04 |
| P-03 | Exactly three availability states; unknown label errors | 01-01 |
| P-04 | Every card field parsed per the selector table | 01-09, 01-10, 01-11, 01-12, 01-13 |
| P-05 | Tri-state histogram is exact on real fixtures | 01-06 |
| P-06 | Location line parsed from the right | 01-13 |
| P-07 | Buddhist date converted to ISO | 01-14 |
| P-08 | Out-of-range page is end-of-data, not an error | 01-08 |
| P-09 | In-range page with zero cards raises | 02-07 |
| P-10 | Stop policy: last / streak / max / full scan | 02-05, 02-06 |
| P-11 | Throttle floor, jitter, retries, `Retry-After`, 403 abort | 02-01, 02-02, 02-03 |
| P-12 | Atomic checkpoint; resume skips done pages | 02-10, 02-11 |
| P-13 | CLI exit codes 0/2/3/4; dry-run makes no requests | 02-13, 02-14, 02-15 |
| P-14 | Only `available` published; others counted | 03-05 |
| P-15 | Duplicate unit ids dropped, first wins | 03-06 |
| P-16 | Grouping merges case variants, recomputes ranges | 03-08, 03-10 |
| P-17 | Deterministic output with a frozen clock | 03-11 |
| P-18 | Dataset invariants machine-checked | 03-12 |
| P-19 | Detail parser: coords, project code, breadcrumb | 04-01, 04-02 |
| P-20 | Out-of-Thailand coordinates rejected | 04-03 |
| P-21 | Coordinate cache: hits, negatives, corruption | 04-05, 04-06, 04-07 |
| P-22 | One detail request per building; warm cache = 0 | 04-09 |
| P-23 | Buildings merged by `project_code` | 04-12, 04-13 |
| P-24 | Live site still parses (opt-in) | 02-16 |

### Web

| Req | What it proves | Task(s) |
|---|---|---|
| W-01 | Loader validates schema/version/shape | 05-04, 05-05 |
| W-02 | Size bounds inclusive, null unbounded, invalid ignored | 05-07 |
| W-03 | Beds exact, Studio = 0, 4+ = ≥ 4 | 05-08 |
| W-04 | District exact, building substring case-insensitive | 05-09 |
| W-05 | Filters apply before the viewport cut | 05-17 |
| W-06 | Null coords never in view, always in no-location | 05-11, 07-10 |
| W-07 | Bubble radius/colour rules | 07-02 |
| W-08 | `moveend` updates the list; `move`/`zoom` do not | 07-05, 07-15 |
| W-09 | Selection sync bubble ↔ row ↔ panel ↔ deselect | 07-09 |
| W-10 | Freshness and partial banner | 05-19 |
| W-11 | Filter bar names, debounce, validation | 06-01, 06-02, 06-03, 06-04 |
| W-12 | Rows show recomputed ranges | 05-10, 06-06 |
| W-13 | Unit links open in a new tab safely | 06-07, 07-16 |
| W-14 | Empty/loading/error states | 05-20, 06-09, 06-12 |
| W-15 | List performance budget | 06-11 |
| W-16 | Tile URL + attribution | 07-01 |
| W-17 | Keyboard reachability, landmarks, live regions | 08-07 |
| W-18 | Mobile layout, no horizontal scroll | 07-12, 08-09 |
| W-19 | No console errors during the main flow | 07-16 |

### Ops

| Req | What it proves | Task(s) |
|---|---|---|
| O-01 | Nightly runs crawl → build → check → publish | 08-02, 08-04 |
| O-02 | Failing `check` blocks publish | 08-04 |
| O-03 | Second concurrent run does not duplicate work | 08-01, 08-04 |
| O-04 | Retention keeps 14 runs / 7 raw dirs | 08-03 |
| O-05 | Empty dataset never published | 03-07, 03-12 |
| O-06 | `make check` passes on a clean clone | 08-06 |

### UI rules

| Rule | Statement | Task(s) |
|---|---|---|
| F1 | Size bounds inclusive; empty = unbounded; invalid ignored | 05-07, 06-01 |
| F2 | Beds exact, Studio = 0, 4+ = ≥ 4 | 05-08, 06-02 |
| F3 | District exact, `All` = no filter | 05-09, 06-02 |
| F4 | Building substring, case-insensitive, trimmed, 200 ms debounce | 05-09, 06-03 |
| F5 | Filters before viewport; visible = ≥ 1 passing unit | 05-17, 06-05 |
| F6 | Null coords never in viewport; listed in no-location | 05-11, 07-10 |
| F7 | Radius from unit count, clamped; selected colour | 07-02 |
| F8 | `moveend` only, list within 200 ms | 07-05, 07-15 |
| F9 | Selection/deselection contract | 05-16, 07-09 |
| F10 | State in React only, no URL sync | 05-14, 05-15, 05-16 |
| F11 | `generatedAt` local time; partial warning | 05-19 |

### Pipeline rules

| Rule | Statement | Task(s) |
|---|---|---|
| P1 | Every network call through `ThrottledFetcher` | 02-01, 02-03 |
| P2 | Zero cards in range is an error | 02-07 |
| P3 | Out-of-range / stub is end-of-data | 01-08, 02-07 |
| P4 | Stop policy: streak, last page, max pages, full | 02-05, 02-06 |
| P5 | Only `available` reaches the dataset | 03-05 |
| P6 | Coordinates once per building, cached, logged | 04-05, 04-09 |
| P7 | Grouping key normalised; merge by `project_code` | 03-08, 04-12 |
| P8 | Deterministic ordering and bytes | 03-10, 03-11 |
| P9 | `check` exit policy | 03-12, 03-14, 08-06 |
| P10 | Every run resumable | 02-10, 02-11 |

---

## 8. Decisions taken while cutting the plan

These are the reconciliations made during task preparation, where `plan/` was ambiguous, internally
inconsistent, or contradicted by the installed toolchain. Implementers follow the task files; this
list explains why they differ from the plan text.

- **Trunk.** `main` (the owner's "master"). Stage branches, pushes and merge commits from
  `plan/91-agent-briefs.md` §5 are not used; every task is one commit on the trunk.
- **URL API is function-based.** `parse_search_url` / `build_search_url` live in `urls.py` (01-03,
  01-04), not `SearchSpec.from_url` / `.to_url` as the plan sketches. Stage 02 calls the functions;
  the deviation avoids a `models.py` ↔ `urls.py` import cycle and is recorded in both stages.
- **`stats.partial` is explicit.** The manifest carries a `partial` flag written by `finish()`
  (02-12); stage 03 copies it and adds `errorCount > 0` / skipped pages. An `empty_streak` stop is a
  normal nightly end, **not** partial — otherwise `check` would refuse every nightly publish. The
  run-mini manifest (03-01) carries `partial: false`.
- **District/building filters live on the building.** A `Unit` has no district or name, so
  `filterUnits` does size + beds and `filterBuilding` does district + name + `filterUnits`.
  `composeView` is therefore `filterBuilding` over buildings → flatten units → `groupByBuilding`
  (recomputed ranges) → `splitByLocation` (05-17).
- **Exit codes** follow the plan's table: crawl `0/2/3/4`; build `0/1/2/4` with an empty selection
  mapped to `1` (data failure), not the fifth code an early draft used; check `0/1/2`. The runbook
  documents 0–4 only (08-11).
- **E2E count.** The plan's 7 scenarios became 15 desktop cases (07-14…07-16) plus 5 mobile cases
  (08-09) = 20; stage 08's release pass expects 20.
- **Playwright bootstrap.** The plan assumed `globalSetup` runs before `webServer`; Playwright does
  the opposite, and `vite preview` serves `dist/`. 07-13 copies the sample dataset into
  `web/public/data/` **and** `web/dist/data/`, refusing to run when `dist/` is missing.
- **Card assembly is one task.** `UnitCard` requires all fields, so 01-05…01-16 are extractors and
  01-17 composes `parse_card` / `parse_search_page` and owns the golden snapshot.
- **`capped` is not `partial`.** An enrichment fetch cap leaves the dataset publishable; only an
  aborted enrichment (or a crawl-level `partial`) sets `stats.partial`, so a nightly run is not
  blocked by a budget guard (04-10, 04-15).
- **A dry run cannot know `total_pages`.** `crawl --dry-run` prints one URL per `--max-pages`
  (default 1) and never fetches; the plan's "exactly 2 URLs" line is unreachable offline and is
  flagged in 02-13.
