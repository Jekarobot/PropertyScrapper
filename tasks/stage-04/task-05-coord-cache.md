---
id: 04-05
stage: 04
title: Persistent building coordinate cache
points: 1
owner: pipeline
status: todo
depends_on: [04-02]
files:
  - pipeline/src/connex_watch/coord_cache.py
  - pipeline/tests/test_coord_cache.py
---

# 04-05 · Persistent building coordinate cache

**Points:** 1 · **Stage:** 04 — Enrichment · **Owner lane:** pipeline · **Depends on:** 04-02 · **Unblocks:** 04-06, 04-07, 04-08, 04-09

## Goal

Create `CoordCache`: a JSON file on disk that maps a normalised building name to a resolved coordinate entry, with atomic save and load.

## Why this task exists

A cold run may need roughly a thousand detail requests; a nightly run must need a few dozen. The difference is this file: coordinates are resolved once per building and reused across runs, which is what makes a daily rebuild affordable (rule P6). It is machine state, not published data — it lives under `data/`, never in the dataset and never in git. The format has to survive schema growth (a later task adds nothing, but a human may inspect it) and must never be the reason a night's run loses a thousand requests, so saving is atomic and keys are the same normalised names stage 03 groups on.

## Current state (preconditions)

- 04-02 done: `detail_parser.py` is complete (coordinates, identity, reasons); nineteen-plus cases green.
- 03-02 done: `pipeline/src/connex_watch/normalize.py` provides `normalize_name`, the single normaliser used for grouping (`The Bangkok Sathorn` → `the bangkok sathorn`).
- 00-01 done: `.gitignore` already lists `data/cache/`, so this file is untracked from the start — do **not** edit `.gitignore` here.
- `coord_cache.py` and `tests/test_coord_cache.py` do not exist; `pytest pipeline -q` is green.

## Read before you start

- `CLAUDE.md` §3 (inject the clock), §5 (one implementation of normalisation), §10 (all output is machine state).
- `plan/stage-04-enrichment.md` Task 4.2 — the exact file format and the six TDD bullets.
- `plan/02-architecture.md` §6 rule P6 — at most one fetch per building per run, cached across runs. `pipeline/src/connex_watch/normalize.py` — the function to reuse, not re-implement.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `pipeline/src/connex_watch/coord_cache.py` | `CacheEntry`, `CoordCache`, file format, atomic save |
| create | `pipeline/tests/test_coord_cache.py` | the seven cases below |

Nothing else may change: no enrichment loop, no CLI flag, no `.gitignore` edit, no dataset change.

## Test first (red)

1. `test_coord_cache.py::test_load_of_absent_file_is_an_empty_cache` — `CoordCache.load(tmp_path / "missing.json")` → `get("the bangkok sathorn") is None`, and the file is **not** created by loading.
2. `...::test_put_then_get_returns_the_entry` — `put` with `lat 13.7187616`, `lng 100.5202356`, `project_code "p00348"`, `name "The Bangkok Sathorn"`, `source_unit_id "CX-84121"`, injected `now` → `get` returns those values, `attempts == 1`, `resolved_at` equals the injected clock.
3. `...::test_keys_are_normalised_names` — `put("  The  Bangkok Sathorn ", …)` then `get("the bangkok sathorn")` hits, and the saved JSON contains the key `the bangkok sathorn` (and no other spelling).
4. `...::test_save_then_load_round_trips_unknown_fields` — a pre-written file with an extra top-level key and an extra field inside an entry → load, save, reload: both extras survive unchanged.
5. `...::test_save_is_atomic` — after `save()` no sibling `.tmp` file remains and the target parses as complete JSON; with `os.replace` monkeypatched to raise, the previous file content is byte-identical afterwards.
6. `...::test_saved_file_has_schema_version_and_updated_at` — `schemaVersion == 1`, `updatedAt` equals the injected clock serialised as ISO-8601 UTC ending in `Z`, `entries` is an object keyed by normalised name.
7. `...::test_default_cache_path_is_under_data_cache` — the module's default path constant is `data/cache/buildings.json`, relative to the repo root, so the existing `data/cache/` ignore rule covers it.

Expected red: `ModuleNotFoundError: No module named 'connex_watch.coord_cache'` for every case.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_coord_cache.py -q`

## Implement (green)

- `CacheEntry` — pydantic v2 model: `lat` (float or None), `lng` (float or None), `project_code`, `name`, `source_unit_id`, `resolved_at` (aware UTC datetime), `attempts` (int, minimum 1), `reason` (str or None, present on negative entries only). JSON keys are exactly `lat`, `lng`, `projectCode`, `name`, `sourceUnitId`, `resolvedAt`, `attempts`, `reason` — same camelCase style as the dataset models from 03-11.
- `CoordCache.load(path, *, retry_after_days=7) -> CoordCache` — a missing file yields an empty cache without creating anything; loading is not a mutation. `retry_after_days` is stored for 04-06 and unused here.
- `get(key) -> CacheEntry | None` and `put(key, *, lat, lng, project_code, name, source_unit_id, reason=None, now) -> None`; both normalise the key with `normalize_name` from `normalize.py` — one normaliser for grouping and cache alike.
- `save()` writes the whole document with `schemaVersion: 1`, `updatedAt` from the last injected `now`, and `entries`; write a sibling temp file, flush, then `os.replace` onto the target. Never write in place, never leave a temp file behind.
- `put` records `attempts = 1` for a new key, `resolved_at = now`. Incrementing and TTL live in 04-06; corruption errors live in 04-07; `len`/`pending` live in 04-08 — do not pre-empt them.
- `now` is always injected (`datetime` argument); no `datetime.now()` inside the module. The default path constant is `data/cache/buildings.json`. One-sentence docstring on the class and on `save`.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_coord_cache.py -q` — seven green.

## Refactor / keep it DRY

There is exactly one serialiser pair (model → JSON keys, JSON → model) used by `save` and `load`; no second dict-bashing path in the tests' helpers. Normalisation is imported, not copied. Timestamp formatting lives in one helper shared with 04-06.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_coord_cache.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
git check-ignore -v data/cache/buildings.json
```

Expected: 7 passed, suite green, `ruff` exits 0, and `git check-ignore` prints the `data/cache/` rule (proof the cache is untracked — no ignore edit needed). Manual check: `pytest` twice produces an identical file when `now` is frozen.

## Out of scope (do not do)

- The negative-cache TTL, corruption errors, `len`/`pending` — 04-06, 04-07, 04-08.
- Any HTTP: this module never imports `httpx` or `fetch`.
- Pruning, compaction, size limits, file locking, or a second cache for source pages. Editing `.gitignore`, adding the cache to the dataset, or touching `models.py`.

## Definition of done

- [ ] All seven cases exist, named as written, and pass.
- [ ] Red run observed (`ModuleNotFoundError`) and one line quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean; `git check-ignore` evidence captured.
- [ ] Only the two files above plus this task file are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
04-05: add the persistent building coordinate cache

- points: 1
- red: test_load_of_absent_file_is_an_empty_cache failed (ModuleNotFoundError: coord_cache)
- add CacheEntry and CoordCache with schemaVersion 1 and normalised keys
- save atomically via a sibling temp file and os.replace
- suite: N passed
```

Files in the commit: the two files above and this task file (status + Result). Nothing else.

## Traps and review notes

- Do not key by the raw display name: `The Bangkok Sathorn` and `THE BANGKOK SATHORN` must hit one entry, or the cache silently misses and the run becomes expensive again.
- `load()` must not create or rewrite the file; a read-only checkout or a first run must not produce an empty cache file that later looks authoritative.
- Do not write the file on every `put`: one `save()` at the end of a run. Atomicity is about crashes, not about durability per entry. Timestamps must be timezone-aware UTC; a naive datetime makes `updatedAt` ambiguous and breaks the round-trip test.
- Reviewers: `grep -n "open(" ` shows one temp-file open plus `os.replace`; `grep -n "normalize_name"` shows the import.

## Plan samples are untrusted

The plan prints a JSON document with the target format. Treat it as a spec of keys and value types, not as a template to paste; verify the model serialisation with the installed pydantic version (`CLAUDE.md` §6) and record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
