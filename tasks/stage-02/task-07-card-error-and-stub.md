---
id: 02-07
stage: 02
title: In-range zero cards is an error; stub ends the walk
points: 1
owner: pipeline
status: todo
depends_on: [02-05]
files:
  - pipeline/src/connex_watch/crawl.py
  - pipeline/tests/test_crawl_stop.py
---

# 02-07 · In-range zero cards is an error; stub ends the walk

**Points:** 1 · **Stage:** 02 — Crawler · **Owner lane:** pipeline · **Depends on:** 02-05 · **Unblocks:** 02-14

## Goal
Separate "the site changed" from "the data ended": a page inside the pager's range that yields zero cards raises `ParseError` for that page, a mid-walk stub ends the walk with `partial=True`, and a pager claiming fewer pages than the page just fetched is overruled by the cards with a warning.

## Why this task exists
Risk R1 in `plan/01-discovery.md` §10 is a class rename: the parser stops matching, every page parses to zero cards, and a naive crawler reports "the cluster ended on page 1" and publishes an empty dataset as good news. Rule P2 exists to prevent exactly that, and this task makes it real while keeping the legitimate end-of-data signals working. The third case covers the opposite lie: the pager says the result has fewer pages than the page we just parsed — then the cards are the truth, and the operator gets a warning, not a crash.

## Current state (preconditions)
- 02-05 is done: the sequential walk, accumulation and the ordered stop decision exist; the `error_stub` branch is present but has no dedicated test.
- Stage 01 exposes `ParseError(component, position=None)` in `search_parser.py` (01-05) and the stub flag (01-08); import that exception, do not define a second one.
- `test_crawl_stop.py` is green with seventeen cases, `FakeFetcher` and `page(body, n, total)`.
- No module logs anything yet in the crawl path; `crawl.py` may start using the stdlib `logging` here.

## Read before you start
- `CLAUDE.md` §3 (no weakened assertions, frozen fixtures), §6, §10.
- `plan/01-discovery.md` §3 (end-of-data table) and §10 risk R1.
- `plan/02-architecture.md` §6 rules P2 and P3.
- `plan/stage-02-crawler.md` Task 2.2 rows 7 and 8 plus the "pager lied" trap; `plan/90-test-matrix.md` rows P-08 and P-09.

## Files to touch
| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/crawl.py` | the zero-card guard, the stub end, the pager warning |
| modify | `pipeline/tests/test_crawl_stop.py` | the four cases below |

Nothing else: no change to `search_parser.py`, `models.py`, `fetch.py` or the fixtures.

## Test first (red)
Synthetic pages are built inside the test by string edits on a real fixture, never stored (`plan/assets/fixtures/README.md`): renaming the card root class token `condo-card` simulates R1 and leaves the pager intact; rewriting the pager's `N Of M` moves a page.

1. `test_crawl_stop.py::test_zero_cards_inside_the_range_raises_parse_error` — page 1 = `search_dev_p1` (`1 Of 5`), page 2 = `search_dev_p2` with the card class renamed (`2 Of 5`) → `ParseError` with `component == "cards"` and `position == 2`, its message containing `2` and `cards`; recorded requests exactly `[1, 2]`, so the walk neither continues nor publishes a short result.
2. `test_crawl_stop.py::test_zero_cards_on_the_first_page_is_an_error` — page 1 with the class renamed (`1 Of 5`) → `ParseError` with `component == "cards"`, `position == 1`; recorded requests exactly `[1]`; a broken first page must never look like "0 available, keep probing".
3. `test_crawl_stop.py::test_error_stub_inside_the_pager_range_ends_the_walk_as_partial` — pages 1 = `search_dev_p1` (`1 Of 5`), 2 = `search_dev_p2` (`2 Of 5`), 3 = `search_error_page` → requests exactly `[1, 2, 3]`, `len(pages) == 2`, `len(cards) == 32`, `stop_reason == "error_stub"`, `partial is True`, page 4 never requested.
4. `test_crawl_stop.py::test_pager_smaller_than_the_requested_page_trusts_the_cards` — page 1 (`1 Of 5`), page 2 = `search_dev_p2` with pager `1 Of 1` → both pages' cards kept (`len(cards) == 32`), requests exactly `[1, 2]`, `stop_reason == "last_page"`, `partial is False`, and exactly one WARNING record (`caplog`) whose message contains `2` and `1`; nothing is dropped.

Expected red symptom: `Failed: DID NOT RAISE ParseError` for case 1, or `assert "empty_streak" == "error_stub"` for case 3. A typo failure is not red.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_crawl_stop.py -q`.

## Implement (green)
- Zero-card guard, evaluated after parsing and before the page is appended: if the page is **not** a stub, has zero cards and `page_no <= total_pages`, raise the stage-01 `ParseError("cards", page_no)` — here `position` carries the page number, which is what 02-14 prints. Never a new exception type, and never catch the parser's own `ParseError`; it subclasses `ValueError`, so no broad `except ValueError` around it.
- The guard runs on page 1 too: a renamed card class fails loudly on the first request.
- Stub handling: a stub with `page_no <= total_pages` ends the walk with `error_stub` and `partial=True`; it is not appended to `pages` and adds no cards. A stub beyond `total_pages` stays 02-05's expected end (`empty_streak`, `partial=False`).
- Pager smaller than the page just fetched with cards present: keep the cards, log one `WARNING` on the module logger naming the page number and the claimed total, then stop with `last_page` / `partial=False`. Do not raise, do not continue silently.
- The warning is a single `logger.warning` call: no logging configuration, no handlers, no formatter.
- One-sentence comment on the guard and the pager check stating the rule and the exception.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_crawl_stop.py -q` — twenty-one green.

## Refactor / keep it DRY
The "in range" test (`page_no <= total_pages`) exists once, used by the guard and the stub branch. The parse-error message contains page and field in the order 02-14's CLI expects. Re-run file and suite.

## Verification (how to test it)
```text
. .venv/bin/activate && pytest pipeline/tests/test_crawl_stop.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 21 passed, the pipeline suite green, `ruff` exits 0. Manual check: none — every page is a fixture or a string edit of one.

## Out of scope (do not do)
- Retrying a zero-card page, or a byte-length heuristic for the stub.
- A "warn and continue" mode, an env var to disable the guard, a downgrade to `empty_streak`.
- Changing the parser's selectors, stub detection or `ParseError` — stage 01 owns them.
- The `--full` interaction with the stub, exit codes, operator messages — 02-06, 02-14.
- Saving the stub or a zero-card page — 02-10. A second warning for a pager claiming too many pages.

## Definition of done
- [ ] All four cases exist, named as written, and pass; the seventeen earlier cases still pass.
- [ ] Red run observed and quoted in the commit body.
- [ ] Target tests + `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `crawl.py`, `test_crawl_stop.py` and this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit
```text
02-07: fail loudly on zero cards and end on a mid-walk stub

- points: 1
- red: test_zero_cards_inside_the_range_raises_parse_error failed (DID NOT RAISE ParseError)
- raise ParseError when a non-stub page inside the pager range has zero cards
- end a mid-walk stub as error_stub/partial and warn when the pager lies low
- suite: 46 passed
```

Files in the commit: `pipeline/src/connex_watch/crawl.py`, `pipeline/tests/test_crawl_stop.py`, and this task file — nothing else.

## Traps and review notes
- Do not decide "zero cards" by document length; the parser already knows what a stub is.
- Do not put the zero-card guard after the stop decision, or the empty-streak rule swallows it and the run exits 0 with an empty result — the exact R1 failure.
- Do not rename the card class in a committed fixture; the synthetic page lives only inside the test, built from the frozen fixture.
- Do not assert a whole warning string with invented punctuation; assert the level, the count and the two facts the operator needs (page 2, claimed total 1).
- Reviewer: case 1 must assert the request list; a guard that raises after fetching page 3 would still raise.

## Plan samples are untrusted
`plan/stage-02-crawler.md` row 8 says "raises `ParseError`" without naming the owning module. Use the type stage 01 created (check `models.py` / `search_parser.py`) and record its location under `## Decisions` if the plan disagrees.

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
