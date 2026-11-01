# Stage 04 — Building enrichment: coordinates + canonical identity

**Goal:** add coordinates to every building that has them, and merge buildings that are the same
project but spelled differently, using the unit detail page — **fetched at most once per building,
cached across runs.**

**Depends on:** Stage 03. **Est:** 50 min. **Agent:** A. **Files:**

```text
pipeline/src/connex_watch/detail_parser.py
pipeline/src/connex_watch/coord_cache.py
pipeline/src/connex_watch/enrich.py
pipeline/tests/test_detail_parser.py
pipeline/tests/test_coord_cache.py
pipeline/tests/test_enrich.py
```

---

## Task 4.1 — Detail-page parser

**Contract**

```python
class DetailInfo(BaseModel):
    lat: float | None
    lng: float | None
    project_code: str | None      # "p00348"
    building_name: str | None     # breadcrumb text
    unit_code: str | None         # "CX-84121"
    reason: str | None            # why coords are missing, for the cache/logs

def parse_detail_page(html: str) -> DetailInfo
```

**TDD order** (`test_detail_parser.py`):

1. `detail_cx84121` → `lat == 13.7187616`, `lng == 100.5202356`, `project_code == "p00348"`,
   `building_name == "The Bangkok Sathorn"`, `unit_code == "CX-84121"`, `reason is None`.
2. Coordinates come from the **first** `center: {lat: …, lng: …}`; the `LatLng(…)` fallback is used
   when `center:` is absent (synthetic fixture built by replacing `center:` with `centre:`).
3. HTML without any coordinate literal → `lat is None`, `lng is None`, `reason == "no-coordinate-literal"`.
4. Coordinates outside Thailand's bounding box (`lat` not in 5.5–20.5, `lng` not in 97–106) →
   `lat/lng = None`, `reason == "out-of-bounds"` (protects against the wrong map block, e.g. a
   street-view widget with default coords).
5. `detail.html` contains the word `"Available"` even for unavailable units — **assert the parser
   never sets a state**, there is no `state` field in `DetailInfo` at all.
6. Malformed/truncated HTML (take the first 2 KB of the fixture) → `DetailInfo` with all `None` and a
   reason, no exception.

**Bounding box constants** live in `detail_parser.py` as named constants with a comment.

Commit: `stage-04: parse detail coordinates and project code`.

---

## Task 4.2 — Coordinate cache

**File format** `data/cache/buildings.json` (committed? **no** — gitignored, it is machine state):

```json
{
  "schemaVersion": 1,
  "updatedAt": "2026-11-01T21:50:00Z",
  "entries": {
    "the bangkok sathorn": {
      "lat": 13.7187616, "lng": 100.5202356,
      "projectCode": "p00348", "name": "The Bangkok Sathorn",
      "sourceUnitId": "CX-84121", "resolvedAt": "2026-11-01T21:50:00Z",
      "attempts": 1
    },
    "some building without coords": {
      "lat": null, "lng": null, "reason": "no-coordinate-literal",
      "attempts": 2, "resolvedAt": "2026-11-01T21:50:00Z"
    }
  }
}
```

**TDD order** (`test_coord_cache.py`):

1. `get("the bangkok sathorn")` on an empty cache → `None`; `put(...)` then `get(...)` returns it.
2. Cache survives a save/load round-trip; unknown extra fields in the file are preserved.
3. **Negative cache**: an entry with `lat is None` is *not* re-fetched until `resolvedAt` is older
   than `retry_after_days=7`; the test injects the clock.
4. Corrupt cache file → `CacheCorruptError` with a "delete the file" hint (never silently reset, that
   would hide data loss and cause a 1000-request storm).
5. `save()` is atomic (`.tmp` + `os.replace`) and keeps entries it does not know about.
6. `len(cache)` and `pending(buildings)` helpers drive the enrichment loop and are unit-tested.

Commit: `stage-04: persistent building coordinate cache`.

---

## Task 4.3 — Enrichment loop

**Contract**

```python
def enrich(buildings: list[Building], fetcher: Fetcher, cache: CoordCache,
           *, max_fetches: int = 1200, max_consecutive_failures: int = 5,
           delay_note: str = "") -> EnrichReport
```

**Rules**

- One candidate unit URL per building: the **cheapest** unit (deterministic) is used as the
  coordinate source, so reruns pick the same unit and the cache stays stable.
- Cache hit → no request. Cache miss → one request → `parse_detail_page` → `cache.put`.
- `FetchError` on one building → cache a negative entry only after the retry policy has exhausted,
  continue with the next building.
- 5 consecutive failures → stop early, mark `EnrichReport.aborted = True`, keep what we have.
- Hard cap `max_fetches` (default 1200) → stop, `EnrichReport.capped = True`.
- Progress log every 25 buildings: `coords ok=… miss=… cached=… remaining=…`.

**TDD order** (`test_enrich.py`, `FakeFetcher` returning the detail fixture for known URLs and
raising for others, injected clock):

1. 3 buildings, empty cache → 3 requests, all 3 with coords; report `{"fetched": 3, "hits": 0, "missing": 0}`.
2. Second run with the warm cache → **0 requests** (the DRY guarantee that makes nightly runs cheap).
3. A building whose detail page has no coordinates → `lat is None`, negative entry cached, report
   `missing: 1`, and the building remains in the list.
4. `max_fetches=1` with 3 buildings → 1 request, `capped is True`, 2 buildings without coords.
5. 5 consecutive `FetchError`s → `aborted is True`, loop stops, already-resolved buildings keep coords.
6. The same building passed twice (title variants already merged) → one request.
7. `FakeFetcher` records the exact URLs → assert they are the detail URLs of the cheapest units.

Commit: `stage-04: enrich buildings with coordinates`.

---

## Task 4.4 — Merge duplicate buildings by project code

**Why:** the title is free text; two spellings of one project become two bubbles otherwise. The
detail page exposes the **canonical** `project_code` (`01-discovery.md` §7).

**Rules**

- After a building is resolved, set `building.project_code` and `building.id = project_code`.
- Any other group with the same `project_code` merges into it: units concatenated, duplicates
  dropped by unit id, `name` = most common original spelling across both groups, ranges recomputed.
- Groups without a code keep the slug id (and a `-2` suffix on collision).
- Report `merged_groups: int` so a spike in title noise is visible.

**TDD order** (`test_enrich.py` continues):

1. Two groups `"The Bangkok Sathorn"` (1 unit) and `"THE BANGKOK SATHORN"` (2 units), both resolving
   to `p00348` → one building `id="p00348"`, `unitCount == 3`, `name == "The Bangkok Sathorn"`.
2. Same merge when one group is cached and one is fetched (order independence).
3. Two different projects with identical names (`id` slug collision) → **no** merge, ids `x` and `x-2`,
   warning logged.
4. Merge preserves `lat/lng` from the cached entry; if the two groups somehow disagree, the first
   non-null wins and a warning is logged.

Commit: `stage-04: merge buildings by project code`.

---

## Task 4.5 — CLI integration

Extend `connex-watch build`:

```text
connex-watch build --raw data/raw/<runId> --out data/runs/<date>/dataset.json
                   [--coords|--no-coords] [--cache data/cache/buildings.json]
                   [--max-coord-fetches 1200] [--dry-run]
```

**TDD order** (`test_cli_build.py`):

1. `--no-coords` → valid dataset with `stats.buildingsWithCoords == 0`, no network, exit 0.
2. `--coords --dry-run` → prints how many buildings would be fetched (cache misses), makes no requests.
3. `--coords` with a fake fetcher → `stats.buildingsWithCoords == 3`, cache file written, exit 0.
4. enrichment aborted → still writes the dataset, `stats.partial == True`, exit code 0 but a warning
   line (a partial map is better than no map; `check` will flag it in Stage 08 policy).
5. `--cache` pointing at a corrupt file → exit 2 with the "delete the file" hint.

Commit: `stage-04: build cli with coordinate enrichment`.

---

## Acceptance criteria

- [ ] `pytest pipeline -q` green; total ≥ 90
- [ ] `build --no-coords` is fully offline and produces the dataset from Stage 03 unchanged (except `stats`)
- [ ] a warm-cache run makes exactly 0 detail requests (assert via fake)
- [ ] dataset has `id` = `p00348` for the Bangkok Sathorn group and `coordSource` set
- [ ] no detail page is fetched twice within one run
- [ ] `ruff check pipeline` clean

## Result

| Task | Tests | Status |
|---|---|---|
| 4.1 | `test_detail_parser.py` (6) | |
| 4.2 | `test_coord_cache.py` (6) | |
| 4.3 | `test_enrich.py` (7) | |
| 4.4 | `test_enrich_merge.py` (4) | |
| 4.5 | `test_cli_build.py` (5) | |

## Traps

- **Never** fetch a detail page per unit. One per building, cheapest unit, cached.
- The detail page is 220 KB raw — never store it, never log it, only parse and discard.
- The Google Maps key in the page is not ours: read numbers only, never call Google APIs.
- Do not use the breadcrumb name as the display name when the group already has a better one — the
  most frequent spelling wins.
- Bounding-box validation is not paranoia: the page has a second map block whose defaults can be
  0,0 on some templates.
