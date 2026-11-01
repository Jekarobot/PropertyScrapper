---
id: 04-10
stage: 04
title: Fetch cap and consecutive-failure abort
points: 1
owner: pipeline
status: todo
depends_on: [04-09]
files:
  - pipeline/src/connex_watch/enrich.py
  - pipeline/tests/test_enrich.py
---

# 04-10 · Fetch cap and consecutive-failure abort

**Points:** 1 · **Stage:** 04 — Enrichment · **Owner lane:** pipeline · **Depends on:** 04-09 · **Unblocks:** nothing yet

## Goal

Bound the enrichment loop: stop after `max_fetches` detail requests with `capped`, and stop after five consecutive `FetchError`s with `aborted`, keeping every coordinate already resolved.

## Why this task exists

The loop's worst case is unbounded: a cache lost on a cold night, or a site that starts refusing mid-run, would produce a thousand requests in one go. Politeness and cost both demand a ceiling that is a number, not a hope. The two limits are different in kind — the cap is a budget, the abort is a "the site is telling us something" signal — and they must be visible in the report, because the dataset built from a stopped run is usable but incomplete and the operator has to know which it was. A single failure must never abort: one broken listing is normal.

## Current state (preconditions)

- 04-09 done: `enrich(buildings, fetcher, cache, *, max_fetches, max_consecutive_failures, now)`, `EnrichReport` with `fetched`/`hits`/`missing`, candidate selection, stamping; seven cases green.
- The loop currently lets a `FetchError` propagate — no failure handling exists yet.
- `FetchError` is defined in `fetch.py` (02-01). `pytest pipeline -q` is green.

## Read before you start

- `CLAUDE.md` §3 (no network; the fake raises), §10 (politeness).
- `plan/stage-04-enrichment.md` Task 4.3 rules 3–5 and its TDD cases 4–5. `plan/02-architecture.md` §6 rule P6 and §7 (cold-run budget ~1100 requests, nightly < 80).
- `plan/01-discovery.md` §8 — the 403/abort reasoning the streak limit mirrors.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/enrich.py` | cap, streak counter, `FetchError` handling, `capped`/`aborted` |
| modify | `pipeline/tests/test_enrich.py` | the six new cases |

Nothing else may change: no CLI, no cache fields, no logging format (04-11), no merge (04-12).

## Test first (red)

1. `test_enrich.py::test_max_fetches_caps_requests` — three cold buildings, `max_fetches=1` → exactly one request, `report.capped is True`, `report.aborted is False`, two buildings with `lat is None`, `missing == 2`.
2. `...::test_hits_do_not_consume_the_fetch_budget` — one warm building plus three cold, `max_fetches=2` → two requests, `hits == 1`, `capped is True`, one cold building left without coordinates.
3. `...::test_five_consecutive_failures_abort_the_run` — nine buildings: 1–3 resolve, 4–8 raise `FetchError`, 9 would succeed → eight requests total, `aborted is True`, `capped is False`, buildings 1–3 keep their coordinates, building 9 is never requested and has none.
4. `...::test_one_fetch_error_does_not_abort` — three buildings, the second raises → `aborted is False`, three requests, the failed building has no coordinates and **no cache entry** (a `FetchError` is not a content miss and is not cached), the other two are stamped.
5. `...::test_failure_streak_resets_after_a_success` — error, success, error, success, error → `aborted is False`, five requests; the streak never reaches five.
6. `...::test_aborted_run_keeps_resolved_coordinates` — after the aborted run, `save()` and reload the cache → the three resolved keys are present with their coordinates, the errored keys are absent.

Expected red: `AttributeError: 'EnrichReport' object has no attribute 'capped'` for cases 1–2 and `Failed: DID NOT RAISE`/the escaped `FetchError` for cases 3–5.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_enrich.py -q`

## Implement (green)

- `EnrichReport` gains `capped: bool` and `aborted: bool`, both default False; 04-11 will pin the full field set and add `merged_groups`.
- Fetch budget: count only actual requests. A cache hit is free and must not consume budget, and a building skipped because the budget is gone must not be requested. When the budget stops the loop, set `capped = True` and stop immediately — do not drain the remaining buildings through the cache.
- Failure policy: on `FetchError` (and only `FetchError`), log/record nothing in the cache, count the building as unresolved, increment a consecutive-failure streak and continue. Any successful request resets the streak to zero. When the streak reaches `max_consecutive_failures` (default 5), set `aborted = True` and stop the loop before the next request.
- Stamping of already-resolved buildings is never undone by either stop. `missing` still means "input buildings without coordinates at the end".
- Both flags can be true only if the cap fired first; the report must not claim `aborted` when only the budget ran out.
- Do not catch broad exceptions: a `KeyError` or pydantic error must still surface.
- One-sentence docstring each on the two flags' meaning.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_enrich.py -q` — thirteen green.

## Refactor / keep it DRY

The "should we make another request?" decision exists once, at the top of the loop body, consulting the budget and the streak; do not re-test the cap in three branches. The stop paths share one exit helper that sets the flag and breaks. `missing` is computed once at the end, not incremented in three places.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_enrich.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 13 passed, whole suite green, `ruff` exits 0. Manual check: `grep -n "max_fetches\|max_consecutive_failures" pipeline/src/connex_watch/enrich.py` shows one comparison each.

## Out of scope (do not do)

- Retry logic: `ThrottledFetcher` already retries and owns backoff; the loop must not retry.
- Treating `BlockedError`/403 specially — the fetcher refuses; no new policy here.
- Wall-clock budgets, sleep/jitter inside the loop, or a `--max-seconds` option. Progress logging and report formatting — 04-11.
- Caching a `FetchError` as a negative entry: the next run must retry it.

## Definition of done

- [ ] All six new cases exist, named as written, and pass; the seven 04-09 cases still pass.
- [ ] Red run observed and one failure line quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean; tests offline.
- [ ] Only the two files above plus this task file are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
04-10: cap enrichment fetches and abort on a failure streak

- points: 1
- red: test_max_fetches_caps_requests failed (AttributeError: capped)
- add max_fetches budget that never charges cache hits
- abort after five consecutive FetchErrors and keep resolved coordinates
- suite: N passed
```

Files in the commit: the two files above and this task file (status + Result). Nothing else.

## Traps and review notes

- Charging a cache hit against `max_fetches` silently turns every warm run into a capped run; case 2 is the guard.
- Caching a `FetchError` would make a transient outage permanent for seven days; the negative cache is for content misses only (detail: `reason` from the parser).
- Continuing to iterate after the cap but skipping requests looks harmless and still costs parse time and hides the stop; break out.
- `aborted` on a single failure makes the nightly build look broken every night; the streak counter, not a counter of totals, decides.
- Reviewers: one `break` per stop path, `except FetchError` only, and no `sleep`.

## Plan samples are untrusted

The plan fixes the defaults (1200 / 5) and the two flag names in prose. Verify the exception name and constructor against `fetch.py` as 02-01 wrote it (`CLAUDE.md` §6); record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
