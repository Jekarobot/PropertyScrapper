---
id: 02-05
stage: 02
title: Page walk with fake fetcher and empty streak
points: 2
owner: pipeline
status: todo
depends_on: [02-04, 01-04, 01-17]
files:
  - pipeline/src/connex_watch/crawl.py
  - pipeline/tests/test_crawl_stop.py
---

# 02-05 · Page walk with fake fetcher and empty streak

**Points:** 2 · **Stage:** 02 — Crawler · **Owner lane:** pipeline · **Depends on:** 02-04, 01-05 · **Unblocks:** 02-06, 02-07, 02-08, 02-13

## Goal
Walk the search pages sequentially — page numbers from our own counter, URLs from the `SearchSpec` — parse each page, accumulate a `CrawlResult`, and stop on the documented policy instead of trusting the pager's "next" link.

## Why this task exists
This is the collector the whole product depends on: no walk, no raw data, no dataset, no map. The walk also decides the project's cost — one request per page, in order, never in parallel — and handles the site's most dangerous property: results are ordered "available first" only as a correlation, so the crawler must keep going for a configurable number of empty pages before concluding the available cluster ended. Too eager misses every available unit; too cautious fetches 546 pages every night.

## Current state (preconditions)
- 02-04 is done: `models.py` exports `StopPolicy` (`max_pages=None`, `empty_streak=2`, `scan_all=False`) and `CrawlResult`.
- Stage 01 is complete (01-04, 01-05, 01-15, 01-16, 01-17): `build_search_url(spec, page_no=n)` from
  `urls.py` builds canonical URLs (the plan's `spec.to_url` sketch is superseded by this function; 01-04 is the
  single URL builder and later stages call it) and `parse_search_page(html) -> PageResult` returns cards, `page_no`, `total_pages`, `total_units` and `is_error_stub`; a missing pager raises `ParseError`.
- `pipeline/src/connex_watch/crawl.py` does **not** exist; `test_crawl_stop.py` has the four 02-04 cases.
- Fixture loaders `search_dev_p1`, `search_dev_p2`, `search_boundary_p71`, `search_error_page` exist (00-05).

## Read before you start
- `CLAUDE.md` §3 (no network in unit tests), §5 (one implementation per rule), §6.
- `plan/01-discovery.md` §3 (pagination and end-of-data) and §6 (ordering is a correlation, the empty-streak rule).
- `plan/02-architecture.md` §6 rules P1–P4.
- `plan/stage-02-crawler.md` Task 2.2; `plan/90-test-matrix.md` rows P-09 and P-10.

## Files to touch
| Action | Path | What goes there |
|---|---|---|
| create | `pipeline/src/connex_watch/crawl.py` | `crawl()` and the loop |
| modify | `pipeline/tests/test_crawl_stop.py` | the fake fetcher and the seven cases below |

Nothing else: no store, no CLI, no logging framework, no change to `fetch.py` or `models.py`.

## Test first (red)
Add a `FakeFetcher`: it maps a page number to an HTML body, records `(page_no, url)` per call, parses the page number out of the URL's `page_no` parameter, and raises `AssertionError` for a page it was not given. A helper `page(body, page_no, total_pages)` rewrites only the pager's `N Of M` text.

1. `test_crawl_stop.py::test_dev_walk_stops_after_an_empty_page_and_the_stub` — pages 1 = `search_dev_p1`, 2 = `search_dev_p2`, 3 = `search_error_page`, default policy → `request_count == 3`, `len(pages) == 2`, `len(cards) == 32`, `state_histogram == {available: 4, check_availability: 6, not_available: 22}`, `stop_reason == "empty_streak"`, `partial is False`.
2. `test_crawl_stop.py::test_dev_walk_requests_each_page_once_in_order` — recorded page numbers exactly `[1, 2, 3]`; recorded URLs exactly `build_search_url(spec, page_no=n)` for those numbers (canonical DEV URL, `page_no` last); no page 4.
3. `test_crawl_stop.py::test_full_spec_walks_the_cluster_then_stops_on_the_streak` — pages 1–70 = `search_dev_p1`, page 71 = `search_boundary_p71`, pages 72–73 = `search_dev_p2`, every pager rewritten to `<n> Of 546` → 73 requests, first page 1, last page 73, `len(pages) == 73`, `state_histogram["available"] == 292` (70 × 4 + 12), `stop_reason == "empty_streak"`, `partial is False`, page 74 never requested.
4. `test_crawl_stop.py::test_single_page_result_stops_after_one_request` — page 1 = `search_dev_p1` with pager `1 Of 1` → exactly 1 request, `len(cards) == 16`, `stop_reason == "last_page"`, `partial is False`.
5. `test_crawl_stop.py::test_scan_all_walks_all_pages_and_ignores_the_streak` — pages 1–3 with pager `<n> Of 3`, page 1 = `search_dev_p1`, pages 2–3 = `search_dev_p2`, `scan_all=True` → 3 requests, `len(cards) == 48`, `state_histogram == {available: 4, check_availability: 7, not_available: 37}`, `stop_reason == "last_page"`.
6. `test_crawl_stop.py::test_walk_counter_is_not_taken_from_the_pager` — every pager claims `1 Of 546` and every page has 4 available cards, `max_pages=4` → recorded page numbers exactly `[1, 2, 3, 4]`; the next number comes from our counter, so no page is fetched twice.
7. `test_crawl_stop.py::test_result_pages_and_cards_are_in_fetch_order` — DEV fake → `pages` page numbers `[1, 2]`, first card id `CX-84121`, last card from page 2, `request_count == 3` while `len(pages) == 2`.

Expected red symptom: `ModuleNotFoundError: No module named 'connex_watch.crawl'`. A typo failure is not red.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_crawl_stop.py -q`.

## Implement (green)
`crawl.py` exposes `crawl(spec, *, fetcher, policy=StopPolicy(), store=None, resume=False) -> CrawlResult`; the loop lives on a `Crawler` object created by `crawl()` (the plan names it), typed against the `Fetcher` protocol, never `ThrottledFetcher`. For `page_no` from 1:

- Build the URL with `build_search_url(spec, page_no=page_no)` and call `fetcher.get(url)` once per page; `request_count` increments per call, including the call that returns the stub.
- Parse with `parse_search_page` (01-17): a parsed page is appended to `pages`, its cards to `cards` in fetch order, and its per-page histogram (`state_histogram` over the card states, 01-06) added bucket-wise to the run histogram. A stub body is neither a page nor a card, and its counts are all `None`.
- `total_pages` comes from the first parsed page's pager and is kept for the run; a later page reporting a different total does not rewrite it.
- The streak counts consecutive parsed pages with **0 available** cards (a page of `check_availability` increments it); one available card resets it.
- Stop decision after each page, first match wins: (1) stub while `page_no <= total_pages` → `error_stub`, `partial=True`; (2) `request_count == max_pages` → `max_pages`, `partial=True`; (3) `streak >= empty_streak` and not `scan_all` → `empty_streak`, `partial=False`; (4) `page_no == total_pages` with ≥1 available card, or `scan_all` → `last_page`, `partial=False`; (5) `page_no == total_pages` with 0 available → probe exactly one page beyond, receive the stub, record `empty_streak`, `partial=False`. Rule 5 is why the DEV walk asks for page 3 of a 2-page result.
- Sequential only: no threads, no asyncio, no futures, one request in flight. `stop_reason` values are the documented strings, never built from an exception message.
- One-sentence docstring on `crawl()` stating the policy, the return type and when it raises.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_crawl_stop.py -q` — eleven green.

## Refactor / keep it DRY
The stop decision is one function returning the reason and the partial flag, reused by 02-06; the URL build, the histogram increment and the stub test each have one call site (the stub test reads the parser's flag, not a byte length). Re-run file and suite.

## Verification (how to test it)
```text
. .venv/bin/activate && pytest pipeline/tests/test_crawl_stop.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 11 passed, the pipeline suite green, `ruff` exits 0. Manual check: none — the fake fetcher is the only transport. The FULL case is a 73-request walk and must stay under ten seconds; slower means the loop re-parses something.

## Out of scope (do not do)
- The `empty_streak=1`, `max_pages` and `error_stub` policy matrix and their warnings — 02-06 / 02-07.
- Duplicate detection — 02-08 (keep duplicates in `cards` here).
- Checkpoints, resume, manifest — 02-09 to 02-12. Logging, `-v`, progress — 02-15.
- Retries and throttling (inside `fetch.py`), detail pages, dataset filtering/grouping — 04-09, 03-05.
- Retrying a page, a second pass, reordering cards, estimating `total_units` from card counts.
- Threads, asyncio or any concurrent fetch — the walk is sequential by contract. A second CLI subcommand.

## Definition of done
- [ ] All seven cases exist, named as written, and pass; the four 02-04 cases still pass.
- [ ] Red run observed and quoted in the commit body.
- [ ] Target tests + `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `crawl.py`, `test_crawl_stop.py` and this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit
```text
02-05: walk search pages with the stop policy

- points: 2
- red: test_dev_walk_stops_after_an_empty_page_and_the_stub failed (ModuleNotFoundError: No module named 'connex_watch.crawl')
- add crawl() with a sequential page walk driven by our own counter
- accumulate CrawlResult and stop on the first matching policy condition
- suite: 36 passed
```

Files in the commit: `pipeline/src/connex_watch/crawl.py`, `pipeline/tests/test_crawl_stop.py`, and this task file — nothing else.

## Traps and review notes
- Never follow the pager's next link or its active page number; the counter is ours. Case 6 fails otherwise, with a page fetched twice.
- The DEV expectation of three requests looks like "one page too many". It is deliberate (rule 5). Do not "fix" the test to two requests.
- 0 available is not 0 cards: a page of `check_availability` increments the streak.
- The FULL fake reuses two bodies, so ids repeat across its 70 pages — realistic, since ranking shifts duplicate units. Duplicates are counted later, not removed here; do not assert `duplicate_units == 0`.
- Reviewer greps `crawl.py` for `httpx`, `time.sleep`, `Thread`, `asyncio`, a second URL-building call site and a second copy of the stop conditions.

## Plan samples are untrusted
The plan sketches `Crawler.run(spec)` and a `while` loop; the repo's `models.py`, `urls.py` and `search_parser.py` signatures win. Check the real `PageResult` field names before coding and record a deviation under `## Decisions` if the plan's field list disagrees.

## Decisions
<!-- append during implementation; leave empty at planning time -->

## Result
| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
