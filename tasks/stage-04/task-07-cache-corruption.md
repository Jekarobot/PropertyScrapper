---
id: 04-07
stage: 04
title: Corrupt cache detection and a safe save
points: 1
owner: pipeline
status: todo
depends_on: [04-05]
files:
  - pipeline/src/connex_watch/coord_cache.py
  - pipeline/tests/test_coord_cache.py
---

# 04-07 · Corrupt cache detection and a safe save

**Points:** 1 · **Stage:** 04 — Enrichment · **Owner lane:** pipeline · **Depends on:** 04-05 · **Unblocks:** nothing yet

## Goal

Fail loudly on a cache file that cannot be trusted: raise `CacheCorruptError` with a "delete the file" hint, never silently reset it, and make saving preserve entries this process never touched.

## Why this task exists

The cache is the only thing standing between a nightly rebuild and ~1000 detail requests. If a half-written or hand-edited file is met with a silent reset, the run looks healthy, spends twenty minutes re-fetching everything, and quietly overwrites the evidence of what went wrong. The correct behaviour is to stop, say which file is broken and what to do about it, and let a human decide. Symmetrically, saving must not truncate knowledge: an entry this run did not touch still belongs to the file.

## Current state (preconditions)

- 04-05 and 04-06 done: format, load/save, `get`/`put`, `needs_fetch`, attempts and the earlier twelve cases are green. A malformed file currently escapes as a raw `json.JSONDecodeError` or a pydantic `ValidationError`.
- `CacheCorruptError` does not exist yet.
- `pytest pipeline -q` is green.

## Read before you start

- `CLAUDE.md` §3 (tests pin behaviour, not exception classes for their own sake) and §9 (failures are stated, never hidden).
- `plan/stage-04-enrichment.md` Task 4.2 case 4 — "never silently reset, that would hide data loss and cause a 1000-request storm".
- `plan/02-architecture.md` §6 rule P6 and §7 (the request budget the error protects).
- `pipeline/src/connex_watch/coord_cache.py` — the load path and the save path you are changing.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/coord_cache.py` | `CacheCorruptError`, validation on load, merge-on-save |
| modify | `pipeline/tests/test_coord_cache.py` | the five new cases, existing cases untouched |

Nothing else may change: no CLI exit code (04-15), no enrichment loop, no fixture edits.

## Test first (red)

1. `test_coord_cache.py::test_corrupt_json_raises_cache_corrupt_error` — a file containing `not json at all` → `CacheCorruptError` whose message contains the file path and the words "delete the file". The file on disk is byte-identical afterwards.
2. `...::test_incompatible_schema_version_is_corrupt` — valid JSON with `"schemaVersion": 2` → `CacheCorruptError`, not a silent migration and not an empty cache.
3. `...::test_entries_that_are_not_an_object_are_corrupt` — `"entries": []` → `CacheCorruptError`.
4. `...::test_entry_validation_failure_is_reported_as_corrupt` — an entry with `"lat": "abc"` → `CacheCorruptError` (the pydantic `ValidationError` must not leak out of `load`).
5. `...::test_save_keeps_entries_it_does_not_know` — a file with an entry for `some building without coords`; this process puts and saves only `the bangkok sathorn`; reload → the untouched entry is unchanged (same values, same `attempts`).

Expected red: `ImportError: cannot import name 'CacheCorruptError'` for cases 1 and 2; cases 3 and 4 currently raise `json`/pydantic errors instead, and case 5 fails only if 04-05's save rewrites the whole document from memory (it must keep the loaded dict).

Run: `. .venv/bin/activate && pytest pipeline/tests/test_coord_cache.py -q`

## Implement (green)

- `CacheCorruptError(Exception)` in `coord_cache.py`, with a message that names the absolute path and ends with a hint telling the operator to delete the file and let the next run rebuild it. The exception carries the path as an attribute so the CLI can print it.
- `load` wraps **only** the file read, the JSON parse and the document validation: missing `schemaVersion`, a `schemaVersion` other than `1`, a non-mapping `entries`, an entry that fails `CacheEntry` validation, and unparseable JSON all raise `CacheCorruptError`. Wrapping a broad `Exception` around the whole method is not acceptable.
- A missing file is still not an error (04-05). An empty `entries` object is valid.
- `save` serialises the in-memory mapping, which was seeded from the loaded document — so keys never touched by this run survive with their values. It stays atomic (sibling temp + `os.replace`).
- No silent repair, no backup copy, no auto-delete: the error is the contract.
- Docstring: one sentence on the exception's meaning and the delete hint.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_coord_cache.py -q` — seventeen green.

## Refactor / keep it DRY

One validation entry point: `load` parses and then calls a single `_validate_document`-style function; the tests' "write a file and load it" helper is reused by cases 1–4 rather than repeated. Do not duplicate the schema-version check in `save`. Re-run after refactoring.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_coord_cache.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 17 passed, whole suite green, `ruff` exits 0. Manual check: write `nope` into a copy of a cache file, call `CoordCache.load` from a shell session, and confirm the message names the path plus the delete hint; the file is still `nope` afterwards.

## Out of scope (do not do)

- The CLI's exit code and message formatting — 04-15.
- Migrating old schema versions, repairing entries, or quarantining a bad file next to the good one.
- File locking, concurrency guards, or a `.bak` rotation.
- Changing the on-disk format or the entry schema.

## Definition of done

- [ ] All five new cases exist, named as written, and pass; the twelve earlier cases still pass.
- [ ] Red run observed and one failure line quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] Only the two files above plus this task file are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
04-07: refuse to load a corrupt coordinate cache

- points: 1
- red: test_corrupt_json_raises_cache_corrupt_error failed (ImportError: CacheCorruptError)
- raise CacheCorruptError with the path and a delete hint
- keep untouched entries when saving
- suite: N passed
```

Files in the commit: the two files above and this task file (status + Result). Nothing else.

## Traps and review notes

- The tempting shortcut is `except Exception: return CoordCache.empty()` — that is the request storm this task prevents. The error must escape `load`.
- Do not let pydantic's `ValidationError` leak: catch it at the entry boundary and re-raise as `CacheCorruptError` with the key name in the message.
- Do not "fix" the file by rewriting a valid document on read. Read paths never write.
- Reviewers: `grep -n "except"` shows narrow handlers only; `grep -n "delete"` shows the hint in one message; `grep -n "UNKNOWN\|default"` shows no schema fallback.

## Plan samples are untrusted

The plan names the exception and the hint in prose only. Keep the exception name exactly as written there so 04-15 and the docs agree; record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
