---
id: 04-08
stage: 04
title: Cache size and pending-buildings helpers
points: 0.5
owner: pipeline
status: todo
depends_on: [04-05]
files:
  - pipeline/src/connex_watch/coord_cache.py
  - pipeline/tests/test_coord_cache.py
---

# 04-08 · Cache size and pending-buildings helpers

**Points:** 0.5 · **Stage:** 04 — Enrichment · **Owner lane:** pipeline · **Depends on:** 04-05 · **Unblocks:** nothing yet

## Goal

Add `len(cache)` and `pending(buildings, now=…)`, the two read-only numbers the enrichment loop and the CLI dry-run need.

## Why this task exists

Two decisions need a count before any request is made: "how big is the cache" (a health line in the log and in the operator's head) and "how many buildings would a `--coords` run actually fetch" (the dry-run output, and the number that tells you the cache was lost). Both must come from the cache's own rules — the same normalisation and the same TTL as `needs_fetch` — or the dry-run would promise a number the real run does not deliver. They are thin, but they are the difference between an operator noticing a broken cache and one discovering it after 1000 requests.

## Current state (preconditions)

- 04-05, 04-06, 04-07 done: `CoordCache` with format, `get`/`put`, `save`, `needs_fetch`, `CacheCorruptError`; seventeen cases green.
- `Building` objects (stage 03, tasks 03-08/03-11) expose `name`; the loop passes a list of them.
- `CoordCache` has no `__len__` and no `pending`; `pytest pipeline -q` is green.

## Read before you start

- `plan/stage-04-enrichment.md` Task 4.2 case 6 and Task 4.3 — where these two numbers are consumed.
- `plan/02-architecture.md` §6 rule P6 — one fetch per building per run.
- `pipeline/src/connex_watch/coord_cache.py` — `needs_fetch` is the rule to reuse.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/coord_cache.py` | `__len__` and `pending` |
| modify | `pipeline/tests/test_coord_cache.py` | the four new cases |

Nothing else may change: no loop, no CLI, no new cache field.

## Test first (red)

1. `test_coord_cache.py::test_len_counts_entries` — a loaded file with two entries → `len(cache) == 2`; after `put` of a third, normalised key → `3`; putting an existing key again does not change the count.
2. `...::test_pending_counts_buildings_that_need_a_fetch` — three building-like objects (`name` only): two warm positives and one unknown → `pending(buildings, now=…) == 1`; all warm → `0`; empty cache and three buildings → `3`.
3. `...::test_pending_ignores_fresh_negatives_and_counts_stale_ones` — a negative entry six days old → not pending; the same entry eight days old → pending. The TTL comes from the cache, not from the test.
4. `...::test_pending_dedupes_by_normalised_name` — `The Bangkok Sathorn` and `THE BANGKOK SATHORN` as two objects, cold cache → `1`, because one request serves both.

Expected red: `TypeError: object of type 'CoordCache' has no len()` for case 1 and `AttributeError: 'CoordCache' object has no attribute 'pending'` for cases 2–4.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_coord_cache.py -q`

## Implement (green)

- `__len__` returns the number of entries currently held, regardless of TTL — it describes the file, not the work.
- `pending(buildings, *, now) -> int` counts the **distinct normalised keys** among `buildings` for which `needs_fetch(key, now=now)` is True. Accept any object exposing `name` (duck-typed); do not import the dataset models into this module and do not add a second normalisation path.
- Both are read-only: no requests, no cache mutation, no logging, no file IO.
- One-sentence docstring on each, stating that `pending` is what a `--coords` run would fetch.
- Do not compute a "would fetch" list of URLs here — the loop owns candidate selection (04-09).

Run: `. .venv/bin/activate && pytest pipeline/tests/test_coord_cache.py -q` — twenty-one green.

## Refactor / keep it DRY

`pending` must call `needs_fetch`; deriving staleness a second time from `resolvedAt` is the duplication this task most invites. The name → key conversion is one helper, already used by `get`/`put`. Re-run after refactoring.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_coord_cache.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 21 passed, whole suite green, `ruff` exits 0. Manual check: `grep -n "def pending" -A 12` shows a call to `needs_fetch` and no date arithmetic.

## Out of scope (do not do)

- Selecting the candidate unit, building URLs, or fetching — 04-09.
- Percentages, hit-rate statistics, or a `stats()` dict; the CLI formats what it needs.
- Pruning or rewriting the file from `pending`.
- Caching the count or memoising it across calls.

## Definition of done

- [ ] All four cases exist, named as written, and pass; the seventeen earlier cases still pass.
- [ ] Red run observed and one failure line quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] Only the two files above plus this task file are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
04-08: add cache length and pending-building helpers

- points: 0.5
- red: test_len_counts_entries failed (TypeError: object of type 'CoordCache' has no len())
- add __len__ and pending(buildings, now=...)
- count distinct normalised keys that need a fetch
- suite: N passed
```

Files in the commit: the two files above and this task file (status + Result). Nothing else.

## Traps and review notes

- Counting objects instead of distinct keys over-reports the request count when two spellings collapse on merge; case 4 is the guard.
- `pending` must honour the negative TTL — a naive `key in entries` check turns every stale negative into a silent permanent skip.
- Do not import `models`/`build_dataset` for a type hint; the duck-typed `name` keeps this module dependency-free.
- Reviewers: no `datetime` arithmetic inside `pending`; one `normalize_name` call path.

## Plan samples are untrusted

The plan mentions both helpers by name and nothing else. Confirm the consumer's expectations in 04-09 and 04-14 before fixing the keyword names; record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
