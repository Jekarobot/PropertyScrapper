---
id: 04-09
stage: 04
title: Enrichment loop with one detail fetch per building
points: 2
owner: pipeline
status: todo
depends_on: [04-05, 03-11, 02-01]
files:
  - pipeline/src/connex_watch/enrich.py
  - pipeline/tests/test_enrich.py
---

# 04-09 · Enrichment loop with one detail fetch per building

**Points:** 2 · **Stage:** 04 — Enrichment · **Owner lane:** pipeline · **Depends on:** 04-05, 03-11, 02-01 · **Unblocks:** 04-10, 04-11, 04-12, 04-14

## Goal

Create `enrich(buildings, fetcher, cache, …)`: for every building, one cache lookup and at most one detail request for its cheapest unit, stamping coordinates onto the building and returning an `EnrichReport`.

## Why this task exists

This is the whole point of stage 04. The search layer gives ~1000 buildings and no positions; the detail layer knows the position of one unit, and every unit in a building shares it. Fetching per unit would multiply the request count by the average units-per-building and earn a block. Fetching per building, cheapest unit first, is both the politeness contract and the cost contract: a cold run is ~1000 requests, a warm run zero (rules P6 and §7's budget). The choice of unit must be deterministic, otherwise two runs pick different source units, the cache keys move, and nothing is reproducible.

## Current state (preconditions)

- 04-05, 04-06, 04-07, 04-08 done: `CoordCache` with `load`, `get`, `put`, `save`, `needs_fetch`, `pending`, `CacheCorruptError`; twenty-one cases green.
- 03-11 done: `build_dataset` produces pydantic `Building` objects with `id`, `name`, `district`, `province`, nullable `lat`/`lng`, `coord_source` and `units[]` (each with `id`, `url`, `price_thb`, `size_sqm`). 02-01 done: the `Fetcher` protocol (`get(url) -> str`) and `FetchError` exist in `fetch.py`; `ThrottledFetcher` is the only network door.
- `pipeline/src/connex_watch/enrich.py` and `tests/test_enrich.py` do not exist. `pytest pipeline -q` is green.

## Read before you start

- `CLAUDE.md` §3 (no network in unit tests, inject the clock), §5 (DRY), §10 (no detail page stored or logged, no Google key). `plan/01-discovery.md` §7 — three units of one building returned identical coordinates; §8 — cost and politeness.
- `plan/02-architecture.md` §3.3 (`coordSource` format `detail:<unitId>`), §6 rules P6/P7, §7 budgets. `plan/stage-04-enrichment.md` Task 4.3 — the contract and its seven TDD bullets.
- `pipeline/tests/fixtures/detail-cx84121.html` — what the `FakeFetcher` returns.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `pipeline/src/connex_watch/enrich.py` | `EnrichReport`, `enrich`, candidate selection |
| create | `pipeline/tests/test_enrich.py` | `FakeFetcher` plus the seven cases below |

Nothing else may change: no CLI, no `build_dataset.py` edit, no cache change, no merge.

## Test first (red)

`FakeFetcher` records every URL and returns the frozen detail fixture for known URLs; all cases inject `now` and build `Building`/unit objects directly (or from `run-mini` via 03-11) — no sockets.

1. `test_enrich.py::test_empty_cache_fetches_one_url_per_building` — three buildings, cold cache → three requests, `fetched == 3`, `hits == 0`, `missing == 0`, and every building has `lat == 13.7187616`, `lng == 100.5202356` and a `coord_source` starting with `detail:`.
2. `...::test_warm_cache_makes_zero_requests` — run twice against the same cache → the second run records **zero** calls, `hits == 3`, `fetched == 0`, same coordinates.
3. `...::test_only_the_cheapest_unit_is_fetched` — a building with units at ฿45 000, ฿40 000 and ฿50 000 → exactly one request, to the ฿40 000 unit's URL; `coord_source` is `detail:<that unit's id>` even though the page's own unit code is `CX-84121`.
4. `...::test_cheapest_unit_tie_breaks_on_unit_id` — two units both at ฿45 000 (`CX-84121`, `CX-135510`) → plain string order picks `CX-135510` (not the numerically smaller id); a second run against a fresh cache picks the same URL.
5. `...::test_missing_coordinates_are_cached_as_a_negative_entry` — the fetcher returns a page with no coordinate literal → the building stays in the list with `lat is None`, `missing == 1`, and the cache holds a negative entry with `reason == "no-coordinate-literal"` and `attempts == 1`.
6. `...::test_fetcher_sees_only_the_candidate_detail_urls` — the recorded URLs equal exactly the three candidate URLs in input order; no `search_en` URL, no URL twice, no second request for a building with three units.
7. `...::test_two_groups_with_the_same_normalised_name_share_one_request` — `The Bangkok Sathorn` and `THE BANGKOK SATHORN`, cold cache → one request, both buildings stamped, `fetched == 1`.

Expected red: `ModuleNotFoundError: No module named 'connex_watch.enrich'`, then `ImportError: cannot import name 'enrich'` once the module is empty.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_enrich.py -q`

## Implement (green)

- `EnrichReport` — a pydantic v2 model (same style as the dataset models) with `fetched`, `hits`, `missing` (ints) for now; 04-10 adds `capped`/`aborted`, 04-11 adds `merged_groups` and the logging.
- `enrich(buildings, fetcher, cache, *, max_fetches=1200, max_consecutive_failures=5, now=None) -> EnrichReport` — processes the list in input order, mutating each `Building` in place (`lat`, `lng`, `coord_source`) and returning only the report. 04-12 will also replace merged groups inside that same list, so callers read the list, not the return value, for buildings. The resolved project code needs no new `Building` field: hold it in a per-run map keyed by the cache key, which 04-12 merges on.
- Candidate unit per building: the unit with the smallest `(price_thb, id)` tuple — one rule, deterministic, no "first on the page".
- Per building: key = `normalize_name(building.name)`; a cache hit stamps from the entry (coordinates, `projectCode`, `coord_source = "detail:" + entry.source_unit_id`) and counts as `hits`; a miss fetches `candidate.url` once, parses with `parse_detail_page`, puts the entry (with `source_unit_id` = the **candidate unit's id**, `name` = the building's display name, `reason` from the parse) and stamps the building.
- `missing` = number of input buildings without coordinates when the loop ends — content misses and anything the caps leave unfetched both count. The same normalised key must not be fetched twice inside one run: the first building stamps, later duplicates hit the now-warm cache (or are skipped as already processed).
- No merging, no failure policy, no progress logging, no printing of URLs' bodies. `now` is a single injected timestamp for the run; when omitted, take the current UTC once at entry. Docstring: one sentence on the one-request-per-building guarantee and the in-place mutation.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_enrich.py -q` — seven green.

## Refactor / keep it DRY

Candidate selection is one function used by the loop and by 04-14's dry-run (which needs the same answer without fetching). Stamping is one helper used by the hit and the miss path — duplicating it is how the two paths drift. The key comes from `normalize_name`; do not re-normalise by hand.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_enrich.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 7 passed, whole suite green, `ruff` exits 0. Manual check: `grep -n "httpx" pipeline/src/connex_watch/enrich.py` prints nothing, and no test leaves a file under `data/`.

## Out of scope (do not do)

- The cap and the consecutive-failure abort — 04-10. Progress logging and report fields — 04-11. Merging by project code — 04-12. CLI flags — 04-14.
- Parallel requests, retries, or a second fetcher: the fetcher already retries and throttles. Caching the raw HTML anywhere, ever. Changing `DetailInfo`, the cache format, or the dataset schema.

## Definition of done

- [ ] All seven cases exist, named as written, and pass; twenty-one cache/parser cases still pass.
- [ ] Red run observed and one failure line quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean; no network in the tests.
- [ ] Only the two files above plus this task file are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
04-09: enrich buildings with one detail fetch each

- points: 2
- red: test_empty_cache_fetches_one_url_per_building failed (ModuleNotFoundError: connex_watch.enrich)
- add enrich() with cheapest-unit candidate selection and warm-cache short circuit
- stamp coordinates and coordSource onto each building
- suite: N passed
```

Files in the commit: the two files above and this task file (status + Result). Nothing else.

## Traps and review notes

- **Never** fetch a detail page per unit: one per building, cheapest unit, cached. The `FakeFetcher` call count in cases 3, 6 and 7 is the assertion that catches this.
- `coordSource` comes from the unit we **requested**, not from the `unit_code` parsed out of the page — case 3 pins the difference deliberately.
- Do not sort buildings to "make it deterministic": input order is the contract, and resorting changes which building wins a cap.
- Do not cache the HTML, log the URL's body, or keep a reference to the response; parse, extract, discard. Do not use the Google key in the page, and do not call any Google API.
- Reviewers: `grep -n "for .*unit"` in the loop body should show candidate selection only; `grep -n "open(\|write"` in `enrich.py` should show nothing. If `Building` turns out to be frozen, replace list items with `model_copy(update=…)` instead of mutating attributes — do not add a mutable side-channel.

## Plan samples are untrusted

The plan's signature also shows a `delay_note` parameter that nothing needs; leave it out and record the omission under `## Decisions`. Verify the `Fetcher`/`FetchError` names against `fetch.py` as 02-01 actually wrote it (`CLAUDE.md` §6).

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
