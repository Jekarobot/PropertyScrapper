# 90 — Test matrix (proof of done)

One row = one requirement → one test → one command. If a row has no test, the requirement is not
implemented. Run `make check` for everything offline; `-m network` and Playwright are opt-in.

Legend: **U** unit (pure, fast), **C** component (jsdom + RTL), **I** integration (files/fixtures),
**E** end-to-end (real browser), **N** network smoke.

---

## Pipeline

| # | Requirement | Level | Test file :: case | Command |
|---|---|---|---|---|
| P-01 | Search URL round-trips losslessly for DEV and FULL specs | U | `test_urls.py::test_roundtrip_dev`, `::test_roundtrip_full` | `pytest pipeline/tests/test_urls.py` |
| P-02 | Canonical parameter order, empty params preserved | U | `test_urls.py::test_canonical_order` | same |
| P-03 | Exactly three availability states, unknown label is an error | U | `test_models.py::test_state_enum` | `pytest -k state` |
| P-04 | Every card field parsed per the selector table | U | `test_search_parser_fields.py` (7 cases) | `pytest -k parser` |
| P-05 | Tri-state histogram of the real fixtures is exact | U | `test_search_parser_states.py::test_histograms` | same |
| P-06 | Location line parsed from the right (commas in names) | U | `test_search_parser_identity.py::test_comma_in_name` | same |
| P-07 | Buddhist date converted to ISO | U | `test_search_parser_identity.py::test_buddhist_date` | same |
| P-08 | Out-of-range page detected as end-of-data, not an error | U | `test_search_parser_pagination.py::test_error_stub` | same |
| P-09 | A card page with 0 cards inside range raises | U | `test_crawl_stop.py::test_zero_cards_is_error` | `pytest -k crawl` |
| P-10 | Stop policy: last page / empty streak / max pages / full scan | U | `test_crawl_stop.py` (10 cases) | same |
| P-11 | Throttle floor, jitter, retries, `Retry-After`, 403 abort | U | `test_fetch.py` (8 cases) | `pytest -k fetch` |
| P-12 | Checkpoint written atomically; resume skips done pages | I | `test_store_resume.py` (7 cases) | `pytest -k store` |
| P-13 | CLI exit codes 0/2/3/4 and dry-run makes no requests | I | `test_cli_crawl.py` (7 cases) | `pytest -k cli` |
| P-14 | Only `available` units published; others counted | I | `test_build_dedupe.py::test_only_available` | `pytest -k dedupe` |
| P-15 | Duplicate unit ids dropped, first occurrence wins | I | `test_build_dedupe.py::test_duplicate` | same |
| P-16 | Grouping merges case variants, recomputes ranges | I | `test_build_group.py` (6 cases) | `pytest -k group` |
| P-17 | Deterministic output (frozen clock, same bytes) | I | `test_build_dataset.py::test_deterministic` | `pytest -k dataset` |
| P-18 | Dataset invariants machine-checked | U | `test_validate.py` (8 violations) | `pytest -k validate` |
| P-19 | Detail parser: coords, project code, breadcrumb, no-state | U | `test_detail_parser.py` (6 cases) | `pytest -k detail` |
| P-20 | Out-of-Thailand coordinates rejected | U | `test_detail_parser.py::test_out_of_bounds` | same |
| P-21 | Coordinate cache: hits, negatives with TTL, corruption | U | `test_coord_cache.py` (6 cases) | `pytest -k cache` |
| P-22 | One detail request per building; warm cache = 0 requests | I | `test_enrich.py::test_warm_cache_no_requests` | `pytest -k enrich` |
| P-23 | Buildings merged by `project_code` | I | `test_enrich_merge.py` (4 cases) | same |
| P-24 | Live site still parses (opt-in) | N | `test_live_smoke.py::test_dev_page` | `pytest -m network` |

## Web

| # | Requirement | Level | Test file :: case | Command |
|---|---|---|---|---|
| W-01 | Dataset loader validates schema/version/shape | U | `loadDataset.test.ts` (6) | `pnpm -C web test loadDataset` |
| W-02 | Size bounds inclusive, null = unbounded, invalid ignored (F1) | U | `filter.test.ts` | `pnpm -C web test filter` |
| W-03 | Beds exact, Studio=0, 4+ = ≥4 (F2) | U | `filter.test.ts` | same |
| W-04 | District exact, building substring case-insensitive (F3/F4) | U | `filter.test.ts` | same |
| W-05 | Filters apply before viewport cut (F5) | I | `App.test.tsx::test_filter_before_viewport` | `pnpm -C web test App` |
| W-06 | Null coords never in viewport, always in `noLocation` (F6) | U | `viewport.test.ts` | `pnpm -C web test viewport` |
| W-07 | Bubble radius/colour rules (F7) | C | `MapView.test.tsx::test_radius` | `pnpm -C web test MapView` |
| W-08 | `moveend` updates the list; `move`/`zoom` do not (F8) | C + E | `MapView.test.tsx::test_moveend_only`, `e2e/app.spec.ts::drag` | `pnpm -C web test MapView`, `pnpm -C web e2e` |
| W-09 | Selection sync: bubble ↔ row ↔ panel ↔ deselect (F9) | C | `App.test.tsx::test_selection`, `BuildingPanel.test.tsx` | `pnpm -C web test` |
| W-10 | Freshness and partial banner (F11) | C | `StatusBar.test.tsx` | same |
| W-11 | Filter bar accessible names, debounce, validation | C | `FilterBar.test.tsx` (10) | same |
| W-12 | Rows show recomputed ranges, not dataset ranges | C | `BuildingRow.test.tsx::test_ranges_from_filtered` | same |
| W-13 | Unit links open in a new tab safely | C + E | `BuildingPanel.test.tsx::test_link`, `e2e/app.spec.ts::popup` | both |
| W-14 | Empty/loading/error states | C | `ResultsList.test.tsx`, `App.test.tsx` | same |
| W-15 | List performance: 200 buildings < 300 ms | C | `ResultsList.test.tsx::test_perf` | same |
| W-16 | Tile URL + attribution (licence) | C | `MapView.test.tsx::test_tiles` | same |
| W-17 | Keyboard reachability, landmarks, live regions | C | `a11y.test.tsx` | same |
| W-18 | Mobile layout: no horizontal scroll, map toggle | E | `e2e/app.spec.ts::mobile` | `pnpm -C web e2e` |
| W-19 | No console errors during the main flow | E | `e2e/app.spec.ts::console` | same |

## Ops

| # | Requirement | Level | Where | Command |
|---|---|---|---|---|
| O-01 | Nightly runs crawl → build → check → publish in order | I | `scripts/test-nightly.sh` | `make test` |
| O-02 | Failing `check` blocks publish | I | `scripts/test-nightly.sh --fail-check` | same |
| O-03 | Second concurrent run exits without duplicating work | I | `scripts/test-nightly.sh --lock` | same |
| O-04 | Retention keeps 14 runs / 7 raw dirs | I | `scripts/test-nightly.sh --retention` | same |
| O-05 | Herd/empty guard: 0 units → exit 1 | I | `test_validate.py::test_empty_dataset` | `pytest -k validate` |
| O-06 | `make check` passes on a clean clone (no dataset) | I | manual | `make check` |

---

## Manual acceptance checklist (release)

- [ ] `make nightly` on a warm cache finishes in < 5 min
- [ ] dataset `stats.partial == false`
- [ ] `buildingsWithCoords / buildings ≥ 0.85`
- [ ] open the built app: 3 rows, ≥ 2 bubbles in the default view, drag updates the list
- [ ] click a bubble: left panel shows units; "view ↗" opens the real site in a new tab
- [ ] resize to 375 px: filter sheet works, map toggles, no horizontal scroll
- [ ] `/data/dataset.json` is < 2 MB

## Anti-patterns this matrix is meant to prevent

| Anti-pattern | Which test catches it |
|---|---|
| Parser silently returns 0 cards | P-09, P-24 |
| "Available" includes "Check availability" | P-05, P-14 |
| A 1000-request storm on every run | P-22 |
| Coordinates fetched per unit | P-22 (`FakeFetcher` call count) |
| Two bubbles for one building | P-16, P-23 |
| Filter shows prices of hidden units | W-12 |
| Map drag filters on every `move` event | W-08 |
| Link opens in the same tab | W-13 |
| Empty dataset published after a site change | P-18, O-05 |
