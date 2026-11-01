---
id: 03-12
stage: 03
title: Dataset invariants
points: 1
owner: pipeline
status: todo
depends_on: [03-11]
files:
  - pipeline/src/connex_watch/validate.py
  - pipeline/tests/test_validate.py
---

# 03-12 · Dataset invariants

**Points:** 1 · **Stage:** 03 — Dataset builder · **Owner lane:** pipeline · **Depends on:** 03-11 · **Unblocks:** 03-14

## Goal

Add `validate_dataset(ds) -> list[str]` returning one human-readable line per broken invariant, and nothing for a healthy dataset.

## Why this task exists

The dataset is produced by a pipeline that can silently degrade: a selector change, a partial page, a broken grouping rule. Structural validation catches the shapes that make the file unusable for the web app — duplicate unit ids (React keys and map selection break), a non-`connex.in.th` url (a link that leaves the product), non-positive price or size (a `0 ฿` bubble), a stale `unitCount`, an inverted price range, a naive `generatedAt` (a browser in another timezone shows the wrong freshness), and a `stats.units` that disagrees with the sum of unit counts (the header lies). The function returns a list rather than raising so the nightly run can print every problem at once; `check` (03-14) turns it into an exit code.

## Current state (preconditions)

- 03-11 is done: `build_dataset.py` exposes `Dataset`, `DatasetStats`, `Building`, `Unit`; `build_dataset` produces a valid run-mini dataset; seven tests pass.
- `pipeline/src/connex_watch/validate.py` does **not** exist.

## Read before you start

- `CLAUDE.md` §3 (TDD), §4 (1 point: 3–8 cases; this task has eight cases: seven violation groups plus the happy path and the multi-fault case), §6.
- `plan/02-architecture.md` §3.3 — the schema and the invariants it implies; §6 P9 (what `check` must reject).
- `plan/stage-03-dataset-builder.md` Task 3.4 item 3 — the exact violation list (eight, in this file's cases).
- `plan/90-test-matrix.md` row P-18 — "dataset invariants machine-checked", 8 violations.
- `pipeline/src/connex_watch/build_dataset.py` — the model and its field names.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `pipeline/src/connex_watch/validate.py` | `validate_dataset` and its checks only |
| create | `pipeline/tests/test_validate.py` | the eight cases below |

Nothing else may change: no CLI (03-14), no build change, no model change.

## Test first (red)

1. `test_good_dataset_has_no_violations` — the run-mini dataset from 03-11 → `[]` (empty list, not `None`, not `"ok"`).
2. `test_duplicate_unit_id_is_reported` — the same unit id in two buildings under two different ids → one violation containing `duplicate unit id` and `CX-1`.
3. `test_unit_url_prefix_is_enforced` — `http://connex.in.th/…` and `https://evil.example/…` each produce a violation containing `url` and the offending url.
4. `test_non_positive_price_or_size_is_reported` — `priceThb` `0` and `-1`, and `sizeSqm` `0`, each give one violation naming the unit id and the offending field.
5. `test_unit_count_mismatch_is_reported` — a building with `unitCount 4` and three units gives a violation naming the building id, `unitCount`, `4` and `3`.
6. `test_inverted_price_range_is_reported` — `priceMin 50000` with `priceMax 45000` gives a violation naming both numbers.
7. `test_naive_generated_at_is_reported` — a dataset whose `generatedAt` has no timezone gives a violation containing `generatedAt`.
8. `test_stats_mismatch_and_multiple_violations` — `stats.units 4` against a sum of `5` gives a `stats.units` violation, and a dataset with three separate faults returns three lines in the documented order, without raising.

Expected red: `ModuleNotFoundError: No module named 'connex_watch.validate'`. A violated model (`ValidationError`) means the test built an impossible dataset — construct bad datasets by mutating a valid dump, not by hand-writing JSON that the model rejects.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_validate.py -q`

## Implement (green)

`validate_dataset(ds: Dataset) -> list[str]` walks buildings and units and appends one single-line message per violation, in this documented order: duplicate unit id → unit url → priceThb → sizeSqm → unitCount → priceMin/priceMax → generatedAt → stats.units. Each message names the offending id and the field, and quotes the bad value (`url http://…`, `priceThb 0`). Prefix every line with the layer (`unit CX-1: …`, `building a-tower: …`, `dataset: …`) so `check` output is greppable. Duplicated ids are reported once per id, not once per extra copy. Empty input structures are not an error here — 0 units is a P9 policy check in 03-14. One sentence of docstring; no printing, no logging, no exit codes.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_validate.py -q` — eight green.

## Refactor / keep it DRY

One check per invariant, each a small function called in the documented order; no giant nested loop with `continue` flags. `stats.units` check reuses the same sum the builder uses (call `sum(b.unitCount)` once). No message string duplicated between two checks.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_validate.py -q
pytest pipeline -q
ruff check pipeline
```

Manual check: `validate_dataset(build_dataset(run_mini, now=…))` prints `[]`; a dump with one price set to `0` prints exactly one line.

## Out of scope (do not do)

- No CLI, no exit codes, no file reading — 03-14.
- No policy checks (0 units, 0 buildings, partial, 48 h freshness) — 03-14 owns P9.
- No coordinate rules: `lat`/`lng` may be `null` at this stage and in stage 04 (03-12 must not require them).
- No schema/JSON-Schema validation, no pydantic re-validation of the whole document.
- No repair, no normalisation, no filtering of bad units.

## Definition of done

- [ ] All eight cases exist, named as written, and pass; red run quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `validate.py` and `test_validate.py`.
- [ ] `validate_dataset` has a one-sentence docstring; each message names the id and the field.
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
03-12: add dataset invariant validation

- points: 1
- red: test_good_dataset_has_no_violations failed (ModuleNotFoundError: connex_watch.validate)
- add validate_dataset: duplicate ids, url prefix, price, size, unitCount, ranges, generatedAt, stats
- return a list of single-line messages in a documented order
- suite: N passed
```

Files in the commit: `validate.py`, `test_validate.py` and this task file. Nothing else.

## Traps and review notes

- Raising instead of returning a list would stop at the first fault and break the nightly log.
- Testing `priceThb <= 0` but not `sizeSqm <= 0` (or the reverse) is the easy half-implementation; both are pinned.
- A naive-datetime check must use `tzinfo is None or utcoffset() is None`; `datetime.utcoffset()` on a naive value raises in some paths.
- Do not count a missing `url` twice (once as url, once as duplicate) — the tests count lines.
- Reviewers: confirm each violation line contains the offending id, and that a valid dataset yields exactly `[]`.

## Plan samples are untrusted

The plan lists the violations in prose but not their messages or order. The message order above is this task's contract; follow it and record the choice under `## Decisions`. Do not add a JSON-Schema dependency because a sample mentions schema validation.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
