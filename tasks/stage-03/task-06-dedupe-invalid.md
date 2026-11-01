---
id: 03-06
stage: 03
title: First-wins dedupe and invalid units
points: 1
owner: pipeline
status: todo
depends_on: [03-05]
files:
  - pipeline/src/connex_watch/build_dataset.py
  - pipeline/tests/test_build_dedupe.py
---

# 03-06 · First-wins dedupe and invalid units

**Points:** 1 · **Stage:** 03 — Dataset builder · **Owner lane:** pipeline · **Depends on:** 03-05 · **Unblocks:** 03-11

## Goal

Drop duplicate unit ids keeping the first occurrence, and drop unusable cards (missing or non-positive price/size) with a warning and a count, so one bad card never fails the whole build.

## Why this task exists

The site's ranking changes between runs, so the same unit can appear on two pages with a different price (`plan/01-discovery.md` §6 says to dedupe by id anyway). The dataset must carry one row per unit, and it must be the *first* occurrence: that is the copy the ranking trusted, and it keeps the output deterministic (P8). Separately, a card whose price or size block failed to parse must not become a `0 ฿` bubble on the map — but it also must not crash the nightly run, because one malformed card out of a thousand is normal. Counting both kinds of loss is what makes a silently degrading crawl visible.

## Current state (preconditions)

- 03-05 is done: `build_dataset.py` has `load_run`, `filter_available`, `UnitSelection` and `select_units`; six tests pass.
- `select_units` currently returns the availability filter only.

## Read before you start

- `CLAUDE.md` §3 (TDD), §5, §6.
- `plan/stage-03-dataset-builder.md` Task 3.2 items 2–3 — duplicate and invalid expectations, and Traps.
- `plan/02-architecture.md` §3.3 `stats.duplicatesDropped`, §6 P5 and P8; §3.3 shows **no** `droppedInvalid` field.
- `plan/90-test-matrix.md` rows P-15 and P-18 (a duplicate id or a non-positive price is a validation failure).
- `pipeline/tests/fixtures/run-mini/pages/page-0002.json` — the duplicate CX-1 at 44000.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/build_dataset.py` | dedupe and invalid-drop steps inside `select_units` |
| modify | `pipeline/tests/test_build_dedupe.py` | the six cases below |

Nothing else may change: no statistics, no grouping, no schema field, no CLI.

## Test first (red)

1. `test_duplicate_unit_id_keeps_first_occurrence` — run-mini yields five units; CX-1's `priceThb` is `45000` (page 1), not `44000` (page 2), and its `position` is still 1.
2. `test_duplicates_dropped_counts_exactly_one` — `UnitSelection.duplicates_dropped == 1` for run-mini, and a run without duplicates reports `0` (not `None`).
3. `test_missing_size_is_dropped_and_counted` — a card with `sizeSqm` null is removed from `units`, `dropped_invalid == 1`, and a WARNING log line names its id (`CX-9`), its page and the field.
4. `test_missing_price_is_dropped_and_counted` — `priceThb` null, `0` and `-1000` are all dropped; `priceThb` `25000` survives (inclusive lower bound of the FULL spec).
5. `test_invalid_card_never_crashes_the_build` — a run containing one null-price card still returns a `UnitSelection` with the other five units intact; no exception type is raised and the warning count is one.
6. `test_invalid_copy_is_removed_before_dedupe` — cards `[CX-1 price 0 on page 1, CX-1 price 44000 on page 2]` give `dropped_invalid == 1`, `duplicates_dropped == 0` and a surviving CX-1 at `44000`.

Expected red: `AttributeError: 'UnitSelection' object has no attribute 'duplicates_dropped'` / `TypeError` from the missing kwarg, and the CX-1 price assertion reading `44000` where `45000` is expected.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_build_dedupe.py -q`

## Implement (green)

Extend `select_units(run)` in this order: availability filter → drop invalid → dedupe by id. Each step is a small private function (`_drop_invalid`, `_dedupe`) so the order is visible in one place.

- Invalid means `priceThb` is `None`, `0` or negative, or `sizeSqm` is `None`, `0` or negative. Each drop logs one WARNING with the unit id, the page number and the offending field name.
- Dedupe keeps the first occurrence in page/position order and counts every later copy in `duplicates_dropped`.
- `UnitSelection` gains `duplicates_dropped: int` and `dropped_invalid: int`, both defaulting to `0`. `dropped_invalid` is **not** published: `plan/02-architecture.md` §3.3 has no such stats field, so it stays on the internal result and in the log. Record that in `## Decisions` if a reviewer expects a field.
- One-sentence docstrings; no new module, no exception class here (03-07 adds one).

Run: `. .venv/bin/activate && pytest pipeline/tests/test_build_dedupe.py -q` — twelve green.

## Refactor / keep it DRY

Validity is one predicate (`_is_usable`), not four inline `is None or <= 0` checks. Dedupe uses one `set` of seen ids plus one output list. Do not re-sort to make dedupe easier: the input order is the first-wins rule.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_build_dedupe.py -q
pytest pipeline -q
ruff check pipeline
```

Manual check: a run over `run-mini` logs no warning (all six available cards are valid) and reports `duplicates_dropped == 1`.

## Out of scope (do not do)

- No `EmptyDatasetError` — 03-07 raises when nothing survives.
- No statistics dictionary, no dataset model, no grouping, no CLI.
- No extra `stats.droppedInvalid` field: the published schema is fixed at `schemaVersion` 1.
- No fuzzy or name-based duplicate detection: ids only.
- No silent drops: every dropped card is counted and logged.

## Definition of done

- [ ] All six cases exist, named as written, and pass; red run quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `build_dataset.py` and `test_build_dedupe.py`.
- [ ] Warning text includes the unit id; the step order is readable in one function.
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
03-06: dedupe by unit id and drop invalid units

- points: 1
- red: test_duplicate_unit_id_keeps_first_occurrence failed (AttributeError: duplicates_dropped)
- add first-wins dedupe and price/size validity drop with one warning per card
- extend UnitSelection with duplicates_dropped and dropped_invalid
- suite: N passed
```

Files in the commit: `build_dataset.py`, `test_build_dedupe.py` and this task file. Nothing else.

## Traps and review notes

- Dropping invalid cards **after** dedupe would let an invalid first copy block a valid later one; the order is pinned by case 6.
- `priceThb == 0` must be invalid: the validator in 03-12 flags `<= 0`, so letting it through only moves the failure.
- Warning once per card, not per field: two warnings for one card makes log counting useless.
- `duplicates_dropped` counts *copies*, so one duplicate id gives `1`, not `2`.
- Reviewers: grep for `<= 0` outside `_is_usable`, and confirm `dropped_invalid` never reaches the dataset model.

## Plan samples are untrusted

The plan describes `dropped_invalid` as "counted" without naming a field, and its statistics sketch omits it. The published schema in `plan/02-architecture.md` §3.3 wins: internal count plus warning, no dataset field. Record it under `## Decisions` when implementing.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
