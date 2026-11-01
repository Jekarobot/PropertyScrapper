---
id: 04-06
stage: 04
title: Negative cache entries with a retry TTL
points: 1
owner: pipeline
status: todo
depends_on: [04-05]
files:
  - pipeline/src/connex_watch/coord_cache.py
  - pipeline/tests/test_coord_cache.py
---

# 04-06 · Negative cache entries with a retry TTL

**Points:** 1 · **Stage:** 04 — Enrichment · **Owner lane:** pipeline · **Depends on:** 04-05 · **Unblocks:** nothing yet

## Goal

Teach the cache when an entry must be fetched again: a positive coordinate never expires, a negative one expires after seven days, and every attempt is counted.

## Why this task exists

Some detail pages simply have no coordinate literal, and a few will be repaired by the site later. Re-fetching them every night is a pointless request storm; never re-fetching them means a fixed page never heals. A TTL on the negative entries is the only rule that satisfies both, and it must be clock-injected so the behaviour is testable without sleeping. `attempts` is the diagnostic: a key with five attempts is a page that keeps failing, and that is worth a human look rather than a sixth try.

## Current state (preconditions)

- 04-05 done: `CacheEntry`, `CoordCache.load`, `get`, `put`, atomic `save`, normalised keys; seven cases green.
- `CoordCache` already stores `retry_after_days` but does not use it yet; `put` always writes `attempts = 1` and `resolved_at = now`.
- `pytest pipeline -q` is green.

## Read before you start

- `CLAUDE.md` §3 — inject the clock; no `datetime.now()` in the module.
- `plan/stage-04-enrichment.md` Task 4.2 case 3 — the negative-cache rule and `retry_after_days = 7`.
- `plan/02-architecture.md` §6 rule P6 and §7 (nightly budget).
- `pipeline/src/connex_watch/coord_cache.py` — the fields and the timestamp helper from 04-05.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/coord_cache.py` | `needs_fetch`, attempt counting, TTL comparison |
| modify | `pipeline/tests/test_coord_cache.py` | the five new cases, existing cases untouched |

Nothing else may change: no loop, no CLI, no corruption handling (04-07).

## Test first (red)

All cases inject `now` as an aware UTC datetime; no test sleeps or reads the wall clock.

1. `test_coord_cache.py::test_positive_entry_never_needs_a_fetch` — an entry with `lat`/`lng` put a year before `now` → `needs_fetch(key, now=…) is False`.
2. `...::test_fresh_negative_entry_needs_no_fetch` — a negative entry (`lat`/`lng` None, reason `no-coordinate-literal`) resolved at `2026-11-01T00:00:00Z`, `now` six days later → `False`.
3. `...::test_negative_entry_at_the_ttl_boundary_needs_a_fetch` — the same entry, `now` exactly seven days later → `True`; one second before the boundary → `False`. The comparison is `>=`.
4. `...::test_put_increments_attempts_and_moves_resolved_at` — `put` twice for the same key with two different `now` values → `attempts == 2`, `resolved_at` equals the second `now`, and the coordinates are the second call's values.
5. `...::test_retry_after_days_is_configurable` — `CoordCache.load(path, retry_after_days=1)` with a negative entry two days old → `True`, while the default-7 cache still says `False`.

Expected red: `AttributeError: 'CoordCache' object has no attribute 'needs_fetch'` for cases 1–3 and 5, and `assert 1 == 2` for case 4.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_coord_cache.py -q`

## Implement (green)

- `needs_fetch(key, *, now) -> bool` — `True` when the key is unknown, or when the entry is negative and `now - resolved_at >= retry_after_days`; `False` for any positive entry regardless of age, and for a negative entry inside the window.
- `put` increments `attempts` when the key already exists (a new key starts at 1) and always sets `resolved_at = now`. A positive entry that overwrites a negative one keeps counting — the counter describes the key, not the outcome.
- `retry_after_days` comes from the constructor/`load` keyword with default `7`; no module-level magic constant in the comparison.
- Time arithmetic on aware UTC datetimes only; one helper converts a stored ISO string to a datetime and it is shared with `save`.
- Docstring: one sentence stating positive entries never expire and negative ones do.
- Do not decide *what* to fetch, do not count missing buildings, do not add `len`/`pending` (04-08) and do not handle corrupt files (04-07).

Run: `. .venv/bin/activate && pytest pipeline/tests/test_coord_cache.py -q` — twelve green.

## Refactor / keep it DRY

The TTL comparison exists once, in `needs_fetch`; 04-08's `pending` must call it rather than re-deriving staleness from `resolved_at`. The ISO parse/format pair stays a single helper. Re-run the file after refactoring.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_coord_cache.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 12 passed, whole suite green, `ruff` exits 0. Manual check: `grep -n "datetime.now\|utcnow" pipeline/src/connex_watch/coord_cache.py` prints nothing.

## Out of scope (do not do)

- Deciding the fetch order, the request cap, or the abort policy — 04-09, 04-10.
- Exponential backoff for negative entries, per-reason TTLs, or a manual "retry failed" flag.
- Pruning expired entries from the file — an expired entry is rewritten by the next `put`. Any `len`/`pending` helper or corrupt-file error.

## Definition of done

- [ ] All five new cases exist, named as written, and pass; the seven 04-05 cases still pass.
- [ ] Red run observed and one failure line quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] Only the two files above plus this task file are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
04-06: expire negative cache entries after seven days

- points: 1
- red: test_fresh_negative_entry_needs_no_fetch failed (AttributeError: needs_fetch)
- add needs_fetch with an injected-clock seven day TTL
- count attempts and move resolvedAt on every put
- suite: N passed
```

Files in the commit: the two files above and this task file (status + Result). Nothing else.

## Traps and review notes

- Do not compare naive datetimes or local time: the file stores UTC with `Z`, so a naive value silently shifts the boundary by hours.
- Boundary direction matters: exactly seven days old is stale, one second less is fresh. Both sides are asserted.
- `attempts` is not an error counter for HTTP failures — an errored fetch never reaches `put` (04-10). Do not reuse it for that.
- Do not add a background "cache cleanup": an expired entry is simply overwritten next time.
- Reviewers: `grep -n "retry_after_days"` shows the parameter and one comparison; `grep -n "timedelta"` shows days only.

## Plan samples are untrusted

The plan states the rule in one line and shows the JSON format with `attempts` and `resolvedAt`. Treat the numbers as the spec, verify the datetime handling against the installed Python and pydantic versions (`CLAUDE.md` §6), and record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
