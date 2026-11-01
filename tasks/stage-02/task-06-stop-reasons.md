---
id: 02-06
stage: 02
title: Last page, max pages, full scan stop reasons
points: 1
owner: pipeline
status: todo
depends_on: [02-05]
files:
  - pipeline/src/connex_watch/crawl.py
  - pipeline/tests/test_crawl_stop.py
---

# 02-06 · Last page, max pages, full scan stop reasons

**Points:** 1 · **Stage:** 02 — Crawler · **Owner lane:** pipeline · **Depends on:** 02-05 · **Unblocks:** nothing

## Goal
Pin the stop-policy matrix: `empty_streak` is configurable, `max_pages` is a hard cap that wins over every other condition, `scan_all` ignores the streak, and the recorded `stop_reason` is one of the documented strings.

## Why this task exists
The default policy is one point in a small decision space, and operators need the others: `--max-pages` for a cheap trial, `--full` for the paranoid weekly scan, a tighter streak when the ranking looks unreliable. Each changes how much of the site we touch and what the manifest says afterwards, so each needs its own proof. The stop reason is also the first thing the nightly log and the runbook look at (stage 08), which makes it a public string, not an internal detail.

## Current state (preconditions)
- 02-05 is done: the walk accumulates `CrawlResult` and has a single stop-decision function with five ordered conditions; `test_crawl_stop.py` is green with eleven cases, `FakeFetcher` and the `page(body, n, total)` helper.
- `StopPolicy` and `CrawlResult` are complete (02-04); `stop_reason` is a plain string.
- No run has ever produced `aborted`, and nothing in stage 02 will.

## Read before you start
- `CLAUDE.md` §3 (one behaviour per test), §4 (a 1-point task: 3–8 cases), §6.
- `plan/01-discovery.md` §6 — why two empty pages by default and what `--full` costs (546 pages).
- `plan/02-architecture.md` §6 rule P4 and §3.2 — the `stopReason` vocabulary.
- `plan/stage-02-crawler.md` Task 2.2 rows 3, 4, 5, 6; `plan/90-test-matrix.md` row P-10.

## Files to touch
| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/crawl.py` | the policy matrix and the reason strings |
| modify | `pipeline/tests/test_crawl_stop.py` | the six cases below |

Nothing else: no store, no CLI, no logging, no new `CrawlResult` field.

## Test first (red)
Reuse `FakeFetcher` and `page()`. Every case asserts the request sequence as well as the reason, so a policy that is read but ignored cannot pass.

1. `test_crawl_stop.py::test_empty_streak_of_one_stops_one_page_earlier` — DEV fake (1 = `search_dev_p1`, 2 = `search_dev_p2`, 3 = `search_error_page`), `empty_streak=1` → recorded pages exactly `[1, 2]`, `len(cards) == 32`, `stop_reason == "empty_streak"`, `partial is False`; page 3 is never requested.
2. `test_crawl_stop.py::test_max_pages_two_stops_after_two_requests` — same fake, `max_pages=2` → exactly `[1, 2]`, `len(pages) == 2`, `stop_reason == "max_pages"`, `partial is True`; the cap wins over the streak (page 2 already made it 1) and over the probe.
3. `test_crawl_stop.py::test_max_pages_one_stops_after_the_first_page` — three-page fake (`<n> Of 3`), `max_pages=1` → exactly `[1]`, `len(cards) == 16`, `stop_reason == "max_pages"`, `partial is True`.
4. `test_crawl_stop.py::test_scan_all_ignores_the_streak` — three-page fake, page 1 with 4 available, pages 2–3 with 0; `empty_streak=1, scan_all=True` → exactly `[1, 2, 3]`, `len(cards) == 48`, `state_histogram == {available: 4, check_availability: 7, not_available: 37}`, `stop_reason == "last_page"`, `partial is False`. Without `scan_all` the same policy stops after page 2.
5. `test_crawl_stop.py::test_max_pages_above_total_does_not_stop_early` — DEV fake, `max_pages=99` → identical to the default: `[1, 2, 3]`, `stop_reason == "empty_streak"`, `partial is False`; a cap is a ceiling, not a target.
6. `test_crawl_stop.py::test_stop_reason_is_one_of_the_documented_strings` — a `CrawlResult` accepts each of `last_page`, `empty_streak`, `max_pages`, `error_stub`, `aborted`; the reasons produced by cases 1–5 are members of that set. `aborted` is accepted but has no producer here.

Expected red symptom: `assert "last_page" == "max_pages"` or a request-count mismatch, because `crawl.py` still reads only the default policy. A `ModuleNotFoundError` means 02-05 is not done.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_crawl_stop.py -q`.

## Implement (green)
- The stop decision keeps its single implementation and gains the matrix in the fixed order `error_stub` (stub inside the pager's range), `max_pages`, `empty_streak` (skipped when `scan_all`), `last_page`. The first match sets both the reason and `partial`: `error_stub` and `max_pages` → `True`; `empty_streak` and `last_page` → `False`.
- `empty_streak` counts consecutive pages with 0 **available** cards; a page of cards with zero `available` still increments it; only a page with at least one available card resets it.
- `max_pages` compares against the page requests issued in this run, including the request that returned the stub, and stops as soon as the cap is reached — never after.
- `scan_all` suppresses only the streak; `max_pages` still applies and the walk still ends at `total_pages`.
- The reason strings are module-level constants or documented on `CrawlResult` and referenced by the loop; never the same literal typed twice.
- No `aborted` producer: a `KeyboardInterrupt` or a surviving `FetchError` leaves the run unfinished — stage 08's business.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_crawl_stop.py -q` — seventeen green.

## Refactor / keep it DRY
Every branch lives in the one stop-decision function; the loop only consumes the returned reason. The reason strings and the partial mapping exist once. Re-run file and suite.

## Verification (how to test it)
```text
. .venv/bin/activate && pytest pipeline/tests/test_crawl_stop.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 17 passed, the pipeline suite green, `ruff` exits 0. Manual check: none — no case makes a request.

## Out of scope (do not do)
- The in-range zero-card error and the dedicated mid-walk stub tests — 02-07 (its branch stays, its tests do not live here).
- The lying-pager warning — 02-07. Duplicate counting — 02-08.
- Resuming, whose pages count towards the streak — 02-11.
- A `max_pages` default, a `--max-pages` flag, any CLI parsing — 02-13.
- Cancellation, `aborted`, signals, timeouts, wall-clock budgets. Logging the reason — 02-15.

## Definition of done
- [ ] All six cases exist, named as written, and pass; the eleven earlier cases still pass.
- [ ] Red run observed and quoted in the commit body.
- [ ] Target tests + `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `crawl.py`, `test_crawl_stop.py` and this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit
```text
02-06: complete the stop policy matrix and reasons

- points: 1
- red: test_max_pages_two_stops_after_two_requests failed (assert 'empty_streak' == 'max_pages')
- make max_pages win over the streak and the last-page probe
- let scan_all ignore the streak while still honouring max_pages
- suite: 42 passed
```

Files in the commit: `pipeline/src/connex_watch/crawl.py`, `pipeline/tests/test_crawl_stop.py`, and this task file — nothing else.

## Traps and review notes
- `max_pages` compares requests, not saved pages; comparing `len(pages)` makes case 2 stop one request late.
- `scan_all` must not disable `max_pages`: an operator asking for both wants the cap.
- Do not derive `partial` from the reason string in two places (the manifest and the CLI read it); the loop sets it once.
- Do not write `aborted` from a bare `except Exception`; a crash must surface, not be relabelled.
- Reviewer: every case must assert the request sequence, not only the reason.

## Plan samples are untrusted
`plan/stage-02-crawler.md` row 5 specifies `max_pages=2` → 2 requests and `max_pages`; the `partial` flags are this task's addition, pinned by the tests. Verify the repo's `CrawlResult` field names before coding and record a deviation under `## Decisions`.

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
