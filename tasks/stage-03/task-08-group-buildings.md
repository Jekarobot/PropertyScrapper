---
id: 03-08
stage: 03
title: Group units into buildings
points: 2
owner: pipeline
status: todo
depends_on: [03-02, 03-03]
files:
  - pipeline/src/connex_watch/build_dataset.py
  - pipeline/tests/test_build_group.py
---

# 03-08 · Group units into buildings

**Points:** 2 · **Stage:** 03 — Dataset builder · **Owner lane:** pipeline · **Depends on:** 03-02, 03-03 · **Unblocks:** 03-09, 03-10

## Goal

Add `group_units(cards)`: collapse selected cards into one group per normalised building name, pick the display name, and recompute unit count and price/size ranges from the units themselves.

## Why this task exists

"Group by condo" is the product: the list, the map bubbles and the left panel are all per building (`plan/02-architecture.md` §5.1). The site gives us units, not buildings, and spells the same building inconsistently, so this is where the dataset's building rows are born. Two rules matter and both are testable: case variants of one name must produce **one** group (P7), and `priceMin`/`priceMax`/`sizeMin`/`sizeMax`/`unitCount` must be computed from the units in the group, never copied from a card or from a previous run — the UI shows ranges that must match the units a user can see.

## Current state (preconditions)

- 03-02 and 03-03 are done: `normalize_name`, `slugify`, `display_name` exist in `normalize.py`.
- 03-05…03-07 are done: `UnitSelection.units` is a list of available, valid, deduped `UnitCard`s.
- `build_dataset.py` has no grouping code; `test_build_group.py` does not exist.

## Read before you start

- `CLAUDE.md` §3 (TDD), §4 (a 2-point task needs 6–12 cases and one integration-level case), §5 (no fuzzy matching).
- `plan/02-architecture.md` §3.1 (Building identity), §3.3 (building fields and ranges), §6 P7.
- `plan/stage-03-dataset-builder.md` Task 3.3 items 1–2 and 6, plus Traps.
- `plan/90-test-matrix.md` row P-16 — the requirement and its six cases.
- `pipeline/tests/fixtures/run-mini/` — CX-1, CX-2, CX-4 are A Tower; CX-3 is B Place; CX-5 is `Condo, A, B`.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/build_dataset.py` | `UnitGroup`, `group_units` |
| create | `pipeline/tests/test_build_group.py` | the eight cases below |

Nothing else may change: no ids (03-09), no location vote or ordering (03-10), no dataset model (03-11).

## Test first (red)

1. `test_case_variants_group_into_one_building` — CX-1, CX-2 (`A Tower`) and CX-4 (`a tower`) yield one group with `unit_count == 3`, key `a tower`, display name `A Tower` (two occurrences beat one).
2. `test_price_range_is_recomputed_from_units` — that group has `price_min == 45000` and `price_max == 50000`; B Place's 32000 never leaks in.
3. `test_size_range_is_recomputed_from_units` — `size_min == 58` (CX-4) and `size_max == 65` (CX-2).
4. `test_single_unit_building` — B Place has `unit_count == 1`, `price_min == price_max == 32000`, `size_min == size_max == 55`.
5. `test_run_mini_groups_to_three_buildings` — `group_units(select_units(load_run(run-mini)).units)` gives exactly three groups with keys `a tower`, `b place`, `condo, a, b` (integration with the reader and the filter).
6. `test_punctuation_variant_merges_but_keeps_display_name` — `["A Tower", "A Tower.", "A Tower"]` gives one group, key `a tower`, display name `A Tower`, `unit_count == 3`.
7. `test_empty_input_gives_no_groups` — `[]` → `[]`; no exception, no placeholder group.
8. `test_different_buildings_stay_separate` — `The Bangkok Sathorn` and `The Bangkok Sathorn Tower` give two groups with keys `the bangkok sathorn` and `the bangkok sathorn tower` (no fuzzy prefix matching).

Expected red: `ImportError: cannot import name 'group_units'` / `UnitGroup`; case 1 failing with two groups (`a tower` twice) is the classic symptom of grouping on the raw string.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_build_group.py -q`

## Implement (green)

`UnitGroup` (dataclass): `key`, `display_name`, `units` (input order), `unit_count`, `price_min`, `price_max`, `size_min`, `size_max`. `group_units(cards) -> list[UnitGroup]`: key each card with `normalize_name(card.buildingName)`, collect in first-appearance order, choose `display_name` from the group's verbatim names via 03-03's helper, and compute the five aggregates with `min`/`max` over the group's units. Empty input returns an empty list. No id, no district, no sorting — later tasks own those. One sentence of docstring; no logging (collisions warn in 03-09).

Fields `lat`, `lng` and `coordSource` are **not** part of this task's data, but the dataset built in 03-11 must carry them as `null` so stage 04 stays purely additive; do not invent placeholder coordinates here.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_build_group.py -q` — eight green.

## Refactor / keep it DRY

Grouping logic exists once, in `group_units`; the web lane's `lib/group.ts` regroups *filtered* units at runtime and must not be mirrored here with different rules — both follow §3.3. Use `defaultdict`/`Counter` plus one small aggregate helper instead of five loops over the same units. `min`/`max` of an empty group must never be evaluated: every group has at least one unit by construction.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_build_group.py -q
pytest pipeline -q
ruff check pipeline
```

Manual check: an ad-hoc call prints three groups; A Tower's three unit ids are `CX-1`, `CX-2`, `CX-4` in page/position order and its ranges are 45000–50000 and 58–65.

## Out of scope (do not do)

- No building ids or collisions (03-09), no district/province vote or ordering (03-10).
- No dataset model, no JSON, no stats, no CLI.
- No fuzzy matching, no prefix merging, no "Tower"/"Condo" suffix stripping.
- No coordinates, no detail-page data, no network.
- No `Building`-shaped dict instead of a dataclass with named fields.

## Definition of done

- [ ] All eight cases exist, named as written, and pass; red run quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `build_dataset.py` and the new `test_build_group.py`.
- [ ] `group_units` and `UnitGroup` each have a one-sentence contract.
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
03-08: group units into buildings

- points: 2
- red: test_case_variants_group_into_one_building failed (ImportError: group_units)
- add UnitGroup and group_units on the normalised name, display name from originals
- recompute unitCount, price and size ranges from the group's units
- suite: N passed
```

Files in the commit: `build_dataset.py`, `test_build_group.py` and this task file. Nothing else.

## Traps and review notes

- Grouping on `card.buildingName` (not the key) is the bug that produces two `A Tower` bubbles; case 1 exists for it.
- Display name must come from the *original* strings; grouping on the key and displaying it puts a lowercased name in the UI.
- Ranges must be recomputed even when a group has one unit (case 4); `min == max` is valid, `None` is not.
- Do not merge `The Bangkok Sathorn` with `The Bangkok Sathorn Tower`: that is fuzzy matching, forbidden by §5 and pinned by case 8.
- Reviewers: grep for `priceMin` assignments outside the aggregate helper, and for any `difflib`/`fuzz` import.

## Plan samples are untrusted

The plan sketches grouping with a `defaultdict` and a dict of ranges, with no display-name rule and no empty-input contract. Follow this task's names and cases; if the sketch disagrees with `plan/02-architecture.md` §3.3, the architecture doc wins. Record deviations under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
