---
id: 02-12
stage: 02
title: Finish manifest, spec-mismatch guard
points: 1
owner: pipeline
status: todo
depends_on: [02-09]
files:
  - pipeline/src/connex_watch/store.py
  - pipeline/src/connex_watch/crawl.py
  - pipeline/tests/test_store_resume.py
---

# 02-12 · Finish manifest, spec-mismatch guard

**Points:** 1 · **Stage:** 02 — Crawler · **Owner lane:** pipeline · **Depends on:** 02-09 · **Unblocks:** 08-02

## Goal
Close a run: `finish(stop_reason, partial)` fills `finishedAt`, `pagesFetched`, `stateHistogram`, `requestCount` and the partial flag from what is actually on disk, and opening an existing run with a different search spec is refused with `ValueError`.

## Why this task exists
A run directory is only trustworthy when the manifest says how it ended. Stage 03 refuses to build from an unfinished or partial run, the nightly script logs the stop reason (08-02), and a human debugging a short run needs `pagesFetched` and `requestCount` at a glance. The spec guard exists because two different searches in one directory would produce a dataset that looks valid and mixes two result sets — impossible to spot afterwards.

## Current state (preconditions)
- 02-09 is done: `RunStore(root, clock=...)`, `create(run_id, spec)` writes `startedAt`, the `spec` parameter block, `stopReason: null` and zeroed counters.
- 02-10 and 02-11 are done: atomic page files, `has_page`/`load_page`, the resumed walk. `test_store_resume.py` has fourteen green cases.
- `finish` does **not** exist; `crawl()` never touches the store after saving pages; nothing counts requests in the store.

## Read before you start
- `CLAUDE.md` §3 (injected clock, no bare `datetime.now()`), §5 (YAGNI), §6.
- `plan/02-architecture.md` §3.2 — the manifest field list and the `stopReason` vocabulary.
- `plan/02-architecture.md` §6 rule P10 and §7 (nightly budget).
- `plan/stage-02-crawler.md` Task 2.3 cases 5 and 7; `plan/90-test-matrix.md` row P-12.

## Files to touch
| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/store.py` | `open`, `record_request`, `finish` |
| modify | `pipeline/src/connex_watch/crawl.py` | record and finish wiring |
| modify | `pipeline/tests/test_store_resume.py` | the five cases below |

Nothing else: no CLI, no dataset code, no change to the page format.

## Test first (red)
Use a clock that returns a fixed value per call (a list consumed in order) so `startedAt` and `finishedAt` are exact strings.

1. `test_store_resume.py::test_finish_sets_the_run_totals` — create `run-a` at `T0`, save pages 1 and 2, record three requests, then `finish("empty_streak", False)` at `T1` → `finishedAt == T1` (ISO UTC `Z`), `stopReason == "empty_streak"`, `partial is False`, `pagesFetched == 2`, `stateHistogram == {available: 4, check_availability: 6, not_available: 22}`, `requestCount == 3`, and the original `runId`/`startedAt`/`spec` unchanged.
2. `test_store_resume.py::test_finish_is_atomic_and_idempotent` — after `finish`, neither `pages/` nor the run directory contains a `*.tmp` file and the manifest parses; a second `finish` with the same arguments leaves exactly one manifest with the same values and no duplicate keys.
3. `test_store_resume.py::test_open_with_a_different_spec_raises_value_error` — create `run-a` with the DEV spec; `RunStore(root).open("run-a", dev_spec)` returns a usable store (`has_page(1)` works), while `open("run-a", FULL_spec)` raises `ValueError` containing `run-a` and `minsize` (55 vs 45). Two searches never share a directory.
4. `test_store_resume.py::test_crawl_finishes_the_manifest` — `crawl(spec, fetcher=dev_fake, store=store)` → the manifest has `stopReason == "empty_streak"`, `partial is False`, `pagesFetched == 2`, `requestCount == 3`, `stateHistogram == {available: 4, check_availability: 6, not_available: 22}` and a non-null `finishedAt`; result and manifest agree.
5. `test_store_resume.py::test_crawl_marks_a_capped_run_partial` — the same fake with `StopPolicy(max_pages=2)` → `stopReason == "max_pages"`, `partial is True`, `pagesFetched == 2`, `requestCount == 2`; a capped run must never look complete to `check`.

Expected red symptom: `AttributeError: 'RunStore' object has no attribute 'finish'`, and `assert None is not None` on `finishedAt` once `finish` exists but is not wired.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_store_resume.py -q`.

## Implement (green)
In `store.py`:
- `open(run_id, spec)` — read the manifest and compare its `spec` block with the requested spec's block (one normalisation function, so a reordered URL is the same search); different → `ValueError` naming the run and the first differing parameter. On success return a store bound to the run with the request counter seeded from the manifest's `requestCount`, so a resumed run keeps the run total.
- `record_request()` — increment the request counter. The crawler calls it once per page request it issues, including the stub request and including resumed runs (the seeded base makes the total correct across invocations).
- `finish(stop_reason, partial)` — compute `pagesFetched` and `stateHistogram` by reading the page files actually present (authoritative and correct after a resume), then rewrite the manifest atomically with `finishedAt` from the clock, `stopReason`, `partial`, `pagesFetched`, `stateHistogram`, `requestCount`; every other key keeps its value and position. A second call with the same arguments is a no-op in effect.
- One-sentence docstring on each method.

In `crawl.py`:
- Call `store.record_request()` immediately before each `fetcher.get(...)`, so a failed request still counts as traffic.
- After the loop decides, call `store.finish(result.stop_reason, result.partial)` before returning — including the stub end and the cap. If the loop raises (`ParseError`, `BlockedError`), do not finish: an unfinished manifest is the correct record of a dead run, and 02-14 maps the exception to an exit code.
- The store stays optional; without one nothing changes for 02-05/02-07/02-11. Never put manifest keys on `CrawlResult` and never let the CLI write the manifest.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_store_resume.py -q` — nineteen green, both other pipeline files still green.

## Refactor / keep it DRY
`create` and `finish` use the same manifest writer; the page scan that computes `pagesFetched` and the histogram reuses 02-11's reader instead of a second JSON parse; the "same spec" comparison uses one canonical URL per side. Re-run the suite.

## Verification (how to test it)
```text
. .venv/bin/activate && pytest pipeline/tests/test_store_resume.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 19 passed, the pipeline suite green, `ruff` exits 0. Manual check: inspect a manifest written under `tmp_path` — exactly the 02-09 keys plus `finishedAt`, `stopReason`, `partial`, `pagesFetched`, `stateHistogram`, `requestCount`; `git status --short` shows no `data/` entry.

## Out of scope (do not do)
- CLI flags, the elapsed summary, exit codes — 02-13 / 02-14.
- A mismatch policy that adopts the requested spec or deletes stale pages.
- A `notes[]` entry for the mismatch, a run history index, a schema migration.
- Finishing a run on an exception; a half-run must stay half-run.
- Dataset-side `partial` handling and the `check` exit policy — 03-12 / 03-14. Retention, pruning, locking.

## Definition of done
- [ ] All five cases exist, named as written, and pass; the 02-09/02-10/02-11 cases still pass.
- [ ] Red run observed and quoted in the commit body.
- [ ] Target tests + `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `store.py`, `crawl.py`, `test_store_resume.py` and this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit
```text
02-12: finish the run manifest and guard the spec

- points: 1
- red: test_finish_sets_the_run_totals failed (AttributeError: 'RunStore' object has no attribute 'finish')
- add open/record_request/finish with totals computed from the pages on disk
- refuse a resume whose search spec differs from the manifest
- suite: 68 passed
```

Files in the commit: `pipeline/src/connex_watch/store.py`, `pipeline/src/connex_watch/crawl.py`, `pipeline/tests/test_store_resume.py`, and this task file — nothing else.

## Traps and review notes
- `pagesFetched` must come from the page files, not an in-memory counter: after a resume the earlier pages were saved by a previous invocation. Case 4 passes either way; a resumed run would not.
- Do not finish in a `finally` block: a run killed by `ParseError` must keep `stopReason: null` so stage 03 refuses it.
- Do not compare specs by object identity, by the raw URL the operator typed, or by dict ordering; compare the normalised parameter blocks. Reviewer: the mismatch message must name the run and the differing parameter, and a second `finish` must not reorder keys.
- Do not double-count the stub request; a second `finish` must not append a note or reorder keys.
- `partial` must stay `False` for `last_page` **and** `empty_streak`, or every nightly run would look incomplete and `check` (03-14) would refuse to publish. The manifest's `partial` key is the single source of truth: 03-11 copies it into `stats.partial` (plus `errorCount`/skipped pages) and must not re-derive it from `stopReason`. Write it even when false, so a reader never has to guess.

## Plan samples are untrusted
`plan/02-architecture.md` §3.2 lists `errorCount` and `notes[]`; this stage writes them only as 02-09's zero/empty values and does not maintain them. Record that deviation under `## Decisions` if the implementer keeps it.

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
