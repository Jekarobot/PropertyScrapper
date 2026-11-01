---
id: 03-10
stage: 03
title: District vote and deterministic ordering
points: 1
owner: pipeline
status: todo
depends_on: [03-08]
files:
  - pipeline/src/connex_watch/build_dataset.py
  - pipeline/tests/test_build_group.py
---

# 03-10 · District vote and deterministic ordering

**Points:** 1 · **Stage:** 03 — Dataset builder · **Owner lane:** pipeline · **Depends on:** 03-08 · **Unblocks:** 03-11

## Goal

Fill each group's `district` / `province` by majority vote over its units (null when none has one) and sort buildings by name then id, units by price then id.

## Why this task exists

`district` drives the district filter (F3) and is displayed on every row, but it is a per-card field: two units of one building can disagree (one card loses its district, one is spelled differently). The dataset has one value per building, so the vote must be explicit and deterministic. Ordering is equally visible: `plan/02-architecture.md` §3.3 and rule P8 require byte-identical output for the same input, which makes the dataset diffable and its snapshots meaningful. Without a pinned order, two identical runs produce different files and every git diff is noise.

## Current state (preconditions)

- 03-08 is done: `group_units` returns `UnitGroup`s with key, display name, units and ranges; 03-09 assigns ids.
- No district/province selection and no sorting exist in `build_dataset.py`.

## Read before you start

- `CLAUDE.md` §3 (TDD), §4 (1-point task: 3–8 cases, negatives included), §6.
- `plan/02-architecture.md` §3.3 — "`buildings[]` sorted by `name` (case-insensitive), then `id`; `units[]` sorted by `priceThb`, then `id`", and "`district` may be `null`"; §6 P8.
- `plan/stage-03-dataset-builder.md` Task 3.3 items 4–5 and Traps (determinism: sort before dumping).
- `plan/01-discovery.md` §2 quirk 2 — a 2-part location line leaves `district` empty, which is why nulls exist.
- `pipeline/tests/fixtures/run-mini/` — A Tower's units are all `Sa Thon` / `Bangkok`.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/build_dataset.py` | `vote_location`, `sort_groups`, `sort_units` |
| modify | `pipeline/tests/test_build_group.py` | the six cases below |

Nothing else may change: no dataset model, no stats, no CLI.

## Test first (red)

1. `test_district_vote_takes_the_most_frequent_non_null` — units with districts `[None, "Khlong Toei", "Khlong Toei"]` vote `Khlong Toei`; run-mini's A Tower gives `Sa Thon` / `Bangkok` for district and province.
2. `test_all_null_location_stays_null` — `[None, None]` → `(None, None)`; a mix of `None` and one value gives that value (nulls never win).
3. `test_vote_tie_breaks_lexicographically` — `["Silom", "Sa Thon"]` (one each) → `Sa Thon`; the result does not depend on unit order (assert on a reversed list too).
4. `test_buildings_sorted_by_name_case_insensitively_then_id` — run-mini's order is `A Tower`, `A, B Condo`, `B Place` (space sorts before comma before `b`); a synthetic pair of groups whose casefolded names are equal is ordered by id.
5. `test_units_sorted_by_price_then_id` — A Tower's units are `[CX-1 (45000), CX-4 (48000), CX-2 (50000)]`; two units at the same price are ordered by id (`CX-2` before `CX-9`).
6. `test_ordering_is_permutation_invariant` — shuffling the input units (seed fixed) produces the same building order and the same unit order, and running twice gives equal results.

Expected red: `ImportError: cannot import name 'vote_location'` and case 4 failing with the first-appearance order (`A Tower`, `B Place`, `A, B Condo`).

Run: `. .venv/bin/activate && pytest pipeline/tests/test_build_group.py -q`

## Implement (green)

- `vote_location(units) -> tuple[str | None, str | None]` — count non-null `district` and `province` separately; the most frequent wins; on a tie the lexicographically smallest string wins (order-independent, unlike "first seen"); no non-null values → `None`.
- `sort_groups(groups) -> list[UnitGroup]` — key `(display_name.casefold(), id)`; returns a new list, no in-place mutation of the caller's data.
- `sort_units(units) -> list[UnitCard]` — key `(priceThb, id)`.
- District/province are per building; `lat`, `lng` and `coordSource` are not set here — 03-11 emits them as `null`.
- One-sentence docstring each; no logging.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_build_group.py -q` — eighteen green.

## Refactor / keep it DRY

One vote helper used for both district and province (no copied loop). Sorting keys live in the sort functions only: no `.sort()` scattered in the assembly task, and never `sorted(..., key=str)` which is case-sensitive. `casefold()` for the building name, plain `min` tie-break inside the vote — both from one place.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_build_group.py -q
pytest pipeline -q
ruff check pipeline
```

Manual check: two consecutive runs over `run-mini` print buildings in `A Tower`, `A, B Condo`, `B Place` order.

## Out of scope (do not do)

- No dataset model, JSON dump, stats or generatedAt — 03-11.
- No sorting inside `group_units` or `assign_building_ids`; the order is applied here, once.
- No district normalisation, translation or whitelist; no fallback to `province` for a missing district.
- No coordinates and no detail-page data.
- No locale-dependent collation: `casefold()` plus code-point comparison only.

## Definition of done

- [ ] All six cases exist, named as written, and pass; red run quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `build_dataset.py` and `test_build_group.py`.
- [ ] The three new functions have one-sentence docstrings; no in-place mutation of caller lists.
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
03-10: vote district and order buildings and units

- points: 1
- red: test_buildings_sorted_by_name_case_insensitively_then_id failed (ImportError: vote_location)
- add vote_location with a lexicographic tie-break and null-safe fallback
- sort buildings by casefolded name then id, units by price then id
- suite: N passed
```

Files in the commit: `build_dataset.py`, `test_build_group.py` and this task file. Nothing else.

## Traps and review notes

- "First seen" tie-breaking makes the output depend on input order; the test reverses the list on purpose.
- Sorting buildings by the raw `name` puts lowercase names after all uppercase ones — use `casefold()`.
- `province` is voted separately from `district`; one vote over tuples loses the province of a district-less unit.
- Sorting units before assigning ids, or after emitting JSON, both break P8; the order is fixed here, last.
- Reviewers: grep for `.sort(` and `sorted(` and check every call site passes the documented key.

## Plan samples are untrusted

The plan states the ordering rule but not the tie-break for the district vote or for equal names. The tie-breaks above are this task's decision; follow them, and record them under `## Decisions` when implementing so the reviewer sees the choice.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
