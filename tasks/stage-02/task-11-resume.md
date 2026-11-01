---
id: 02-11
stage: 02
title: Resume skips completed pages
points: 1
owner: pipeline
status: todo
depends_on: [02-10]
files:
  - pipeline/src/connex_watch/crawl.py
  - pipeline/src/connex_watch/store.py
  - pipeline/tests/test_store_resume.py
---

# 02-11 · Resume skips completed pages

**Points:** 1 · **Stage:** 02 — Crawler · **Owner lane:** pipeline · **Depends on:** 02-10 · **Unblocks:** 02-13, 08-02

## Goal
Make `crawl(..., resume=True)` read the pages already on disk, skip them in the network loop, and return a result containing the cards from every page of the run as if it had never been interrupted.

## Why this task exists
A 546-page crawl takes ~22 minutes and the nightly run can be interrupted by a reboot, a `SIGTERM` or a block. Without resume every restart begins at page 1, which is slow and impolite: refetching pages we already have is exactly the traffic the throttle exists to avoid. `plan/02-architecture.md` rule P10 makes this a hard requirement, and the nightly script (08-02) always passes `--resume`.

## Current state (preconditions)
- 02-10 is done: `save_page` writes `page-<nnnn>.json` atomically and the page ↔ JSON mapping has a reader-side partner ready; `test_store_resume.py` has eight green cases.
- `crawl()` accepts `store=None` and `resume=False` but ignores both.
- `RunStore` knows its run directory; no `has_page` / `load_page` exists.
- The DEV fake in `test_crawl_stop.py` documents the expected DEV walk: pages 1, 2, then the stub.

## Read before you start
- `CLAUDE.md` §3 (determinism, no network in unit tests), §5 (a rule has one implementation), §6.
- `plan/02-architecture.md` §3.2 ("existing file ⇒ page already done ⇒ `--resume` skips it") and §6 rule P10.
- `plan/stage-02-crawler.md` Task 2.3 case 6 and the trap "`--resume` must still re-read the manifest to know `total_pages`".
- `plan/90-test-matrix.md` row P-12.

## Files to touch
| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/store.py` | `has_page`, `load_page` |
| modify | `pipeline/src/connex_watch/crawl.py` | load-then-continue in the loop |
| modify | `pipeline/tests/test_store_resume.py` | the six cases below |

Nothing else: no manifest change, no CLI flag, no new file format.

## Test first (red)
1. `test_store_resume.py::test_has_page_is_false_before_anything_is_saved` — a fresh `create("run-a", spec)` → `has_page(1)`, `has_page(2)`, `has_page(3)` all `False`.
2. `test_store_resume.py::test_has_page_is_true_only_for_saved_pages` — save page 1 → `has_page(1)` is `True`, `has_page(2)` is `False`; numbers are compared as integers, so the four-digit file name cannot fool it.
3. `test_store_resume.py::test_load_page_of_a_missing_page_returns_none` — `load_page(7)` is `None` and does not raise, even when `pages/` does not exist yet.
4. `test_store_resume.py::test_load_page_round_trips_a_saved_page` — save page 1, then `load_page(1)` returns a `PageResult` equal to what was saved: page number, totals, stub flag and a card-for-card comparison including ids, states and positions.
5. `test_store_resume.py::test_resume_fetches_only_the_missing_pages` — save pages 1 and 2 (parsed from `search_dev_p1`, `search_dev_p2`), then `crawl(spec, fetcher=fake, store=store, resume=True)` where the fake only knows the stub for page 3 → recorded exactly `[3]`, `result.request_count == 1`, `len(result.pages) == 2`, `len(result.cards) == 32`, `state_histogram == {available: 4, check_availability: 6, not_available: 22}`, `stop_reason == "empty_streak"`, `partial is False`.
6. `test_store_resume.py::test_resume_does_not_rewrite_completed_pages` — the bytes of `page-0001.json` are identical before and after the resumed crawl; a page that was not fetched is not re-saved.

Expected red symptom: `AttributeError: 'RunStore' object has no attribute 'has_page'` for the first three, `assert [1, 2, 3] == [3]` for case 5 once the methods exist.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_store_resume.py -q`.

## Implement (green)
In `store.py`:
- `has_page(page_no)` — `True` iff this run's page file for that number exists; integer in, boolean out, no parsing, no side effects.
- `load_page(page_no)` — read and rebuild the `PageResult` through 02-10's mapping; `None` when the file is missing. A file that exists but is corrupt raises the underlying error: a corrupt checkpoint must never look like "not done".

In `crawl.py`:
- With `resume=True` and a store, walk `n = 1, 2, ...` while `has_page(n)`, load each page in order, append it to `pages` and its cards to `cards`, add its state counts to the histogram and count its available cards into the streak — exactly as a freshly fetched page would. The first missing `n` is the first page to fetch.
- Each page fetched in this invocation is saved through `store.save_page` **after** the zero-card guard and the stub check: never a broken page, never the stub.
- `result.request_count` counts requests issued by this invocation; pages restored from disk were not requested now (the manifest total is 02-12's business).
- Do not re-save a loaded page, and do not fetch page 1 to confirm the pager: `total_pages`, the streak and the last-page state come from the loaded pages.
- With `resume=False` and a store, start at page 1 and let `save_page` overwrite; never read the disk. Without a store, behave exactly as in 02-05.
- One-sentence docstring on both store methods and a comment on the resume block.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_store_resume.py -q` — fourteen green, and `pytest pipeline/tests/test_crawl_stop.py -q` stays green.

## Refactor / keep it DRY
The "a page is a page" accumulation (append, histogram, streak) is one helper used by both the loaded and the fetched path; duplicating it is how resume starts disagreeing with a fresh run. The existence rule lives in `store.py`, not in the loop. Re-run both files.

## Verification (how to test it)
```text
. .venv/bin/activate && pytest pipeline/tests/test_store_resume.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 14 passed, the pipeline suite green, `ruff` exits 0. Manual check: none — the resumed run still uses the fake fetcher.

## Out of scope (do not do)
- The manifest's `pagesFetched`/`requestCount`/`stopReason` and the spec-mismatch guard — 02-12.
- `--resume` as a CLI flag, choosing a run id, discovering the latest run — 02-13.
- Skipping a range, a partial page (1 and 3 on disk, 2 missing), or checksum re-validation.
- Verifying that saved pages belong to the requested spec — 02-12 does that once, at open time.
- Compressing, pruning or migrating old run directories (non-goal). Concurrency of any kind.

## Definition of done
- [ ] All six cases exist, named as written, and pass; the 02-09/02-10 cases and the crawl-stop file still pass.
- [ ] Red run observed and quoted in the commit body.
- [ ] Target tests + `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `store.py`, `crawl.py`, `test_store_resume.py` and this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit
```text
02-11: resume a crawl from saved pages

- points: 1
- red: test_resume_fetches_only_the_missing_pages failed (assert [1, 2, 3] == [3])
- add has_page/load_page and start the loop after the last saved page
- restore cards, histogram and streak from disk without re-saving anything
- suite: 63 passed
```

Files in the commit: `pipeline/src/connex_watch/store.py`, `pipeline/src/connex_watch/crawl.py`, `pipeline/tests/test_store_resume.py`, and this task file — nothing else.

## Traps and review notes
- Do not start at 1 and skip inside the loop after fetching: the point is not to issue the request. Case 5 asserts the recorded pages.
- Do not ignore loaded pages when computing the streak: a resumed run whose last page is empty must behave like a fresh one (DEV still probes page 3).
- Do not return `None` for a corrupt file; only a missing file is `None`, or a corrupt checkpoint becomes a skipped page.
- Do not save before the parsers' guards accept a page, or a renamed card class leaves a zero-card checkpoint behind.
- Reviewer: both paths must share one accumulation helper, and case 6 must compare bytes, not mtimes.

## Plan samples are untrusted
`plan/stage-02-crawler.md` says `--resume` "must still re-read the manifest to know `total_pages`"; this task reads `total_pages` from the loaded pages instead, the only source the manifest lacks. Record that deviation under `## Decisions` if the implementer keeps it.

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
