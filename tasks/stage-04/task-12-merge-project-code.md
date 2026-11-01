---
id: 04-12
stage: 04
title: Merge buildings by project code
points: 2
owner: pipeline
status: todo
depends_on: [04-09]
files:
  - pipeline/src/connex_watch/enrich.py
  - pipeline/tests/test_enrich_merge.py
---

# 04-12 · Merge buildings by project code

**Points:** 2 · **Stage:** 04 — Enrichment · **Owner lane:** pipeline · **Depends on:** 04-09 · **Unblocks:** 04-13

## Goal

After coordinates are resolved, fold every group sharing a `project_code` into one building: canonical id, concatenated units, most common spelling as the name, recomputed ranges.

## Why this task exists

Stage 03 groups by normalised title, which is free text. Two spellings of one project survive as two buildings only when their normalised forms differ — and even then the map shows two bubbles for one tower. The detail page supplies the project code, which is the site's real identity for the building, so after enrichment there is finally something trustworthy to merge on. This is also what makes the dataset's `id` stable across runs: `p00348` today and `p00348` next month, whatever the title says. Rule P7 in the architecture is this task, and without it the "one bubble per building" promise is only approximately true.

## Current state (preconditions)

- 04-09 and 04-10 done: `enrich` selects the cheapest unit, resolves through the cache, stamps `lat`/`lng`/`project_code`/`coord_source`, honours the cap and the abort; `EnrichReport` has `fetched`/`hits`/`missing`/`capped`/`aborted` (04-11 also adds `merged_groups`, defaulting to 0) and the progress log.
- Stage 03 helpers exist: `display_name(variants)` in `normalize.py`, range recomputation in `build_dataset.py`, slug-based ids with a `-2` collision suffix. `pipeline/tests/test_enrich_merge.py` does not exist; `pytest pipeline -q` is green.

## Read before you start

- `plan/stage-04-enrichment.md` Task 4.4 — the merge rules and the four TDD bullets.
- `plan/02-architecture.md` §3.1 (`Building` identity = `project_code` when known) and §6 rule P7; §3.3 for the output ordering (`name`, then `id`; units by `priceThb`, then `id`).
- `plan/01-discovery.md` §7 — the canonical code and name come from the breadcrumb. `pipeline/src/connex_watch/normalize.py` and `build_dataset.py` — `display_name` and the range helper to reuse.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/enrich.py` | merge by project code, id and name rules, ranges, `merged_groups` |
| create | `pipeline/tests/test_enrich_merge.py` | the seven cases below |

Nothing else may change: no `build_dataset.py` edit, no CLI, no cache change. Import stage-03 helpers even if they are module-private — copying them is worse than importing them.

## Test first (red)

Build the groups directly (they are the merged output of stage 03); all values are from the frozen search fixture: `CX-84121` ฿45 000 / 65 sqm, `CX-135510` ฿45 000 / 68 sqm, `CX-174761` ฿50 000 / 60 sqm.

1. `test_enrich_merge.py::test_groups_sharing_a_project_code_merge_into_one` — `The Bangkok Sathorn` (1 unit) and `THE BANGKOK SATHORN` (2 units), both resolving to `p00348` → one building, `id == "p00348"`, `unit_count == 3`, `name == "The Bangkok Sathorn"`.
2. `...::test_duplicate_unit_ids_are_dropped` — the same unit id present in both groups → exactly one copy, the first occurrence's values win.
3. `...::test_ranges_are_recomputed` — merged building: `price_min == 45000`, `price_max == 50000`, `size_min == 60`, `size_max == 68`.
4. `...::test_merged_building_takes_coordinates_and_coord_source` — `lat == 13.7187616`, `lng == 100.5202356`, `coord_source == "detail:CX-84121"` when the cached entry came from `CX-84121`.
5. `...::test_merge_is_order_independent_with_one_group_cached` — group A's key pre-cached with `projectCode p00348` and coordinates, group B fetched fresh → both input orders give the same id, name, unit count and coordinates, with exactly one request.
6. `...::test_group_without_a_project_code_keeps_its_slug_id` — a third building whose page yields no code keeps its stage-03 id and is not merged into `p00348`.
7. `...::test_report_counts_merged_groups` — the two-group case reports `merged_groups == 1`; a run with nothing to merge reports `0`.

Expected red: `ModuleNotFoundError` for the new test module's subject only if `enrich` is missing, otherwise `AssertionError: assert 3 == 1` on the building count (no merge happens yet).

Run: `. .venv/bin/activate && pytest pipeline/tests/test_enrich_merge.py -q`

## Implement (green)

- After a building is resolved, the run's code map (04-09) says which group has which `p00348`-style code; **replace the list** so that all groups with the same non-null code become one building: units concatenated in input order, duplicates by unit `id` dropped (first wins), `name` = `display_name` over the original spellings of all merged groups, `id` = the project code, `district`/`province` recomputed by `vote_location`. `Building` has no `project_code` field and must not gain one.
- Ranges (`unit_count`, `price_min`, `price_max`, `size_min`, `size_max`) are recomputed from the merged unit list with the same `min`/`max` rule `group_units` applies — never summed, never carried over.
- Coordinates: the merged building takes the first non-null `lat`/`lng` and the matching `coord_source`; when a group was already cached and another was fetched, the result is identical regardless of which came first in the list.
- Groups with no code are untouched and keep their slug id (03-09 already made them unique with a `-2` suffix).
- Order: because `enrich` mutates the list it is given, replace its contents in place (`buildings[:] = merged`) so callers see one building per project.
- `report.merged_groups` = number of groups folded away (`groups_before - groups_after`).
- Docstring: one sentence on the merge key and the in-place replacement.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_enrich_merge.py -q` — seven green.

## Refactor / keep it DRY

The name choice is `display_name`, the ranges are the stage-03 recompute, the district vote is the stage-03 vote: import all three; do not re-derive "most common spelling" with a local `Counter`. The merge loop runs once over the buildings, building an id → group map, not a nested scan per building. Deterministic order comes from the input list plus one explicit sort by the dataset rule.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_enrich_merge.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 7 passed, whole suite green, `ruff` exits 0. Manual check: `grep -n "Counter\|most_common" pipeline/src/connex_watch/enrich.py` prints nothing — the vote belongs to `normalize.py`.

## Out of scope (do not do)

- Same-name groups with **different** project codes — 04-13 (including its warnings).
- Fuzzy name matching, transliteration, or merging on geographic proximity. Re-crawling, re-grouping, or changing stage-03's grouping key.
- Changing the dataset schema, `stats.units`/`stats.buildings`, or `duplicatesDropped` (that counter is a search-layer counter).

## Definition of done

- [ ] All seven cases exist, named as written, and pass; the enrich/cache/parser suites still pass.
- [ ] Red run observed and one failure line quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] Only the two files above plus this task file are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
04-12: merge buildings by project code

- points: 2
- red: test_groups_sharing_a_project_code_merge_into_one failed (assert 3 == 1)
- fold same-project groups into one building with id = project code
- reuse stage-03 display_name and range recomputation, count merged_groups
- suite: N passed
```

Files in the commit: the two files above and this task file (status + Result). Nothing else.

## Traps and review notes

- Do not use the breadcrumb name as the display name when the group already has a better spelling: the most frequent original wins (case 1 asserts `The Bangkok Sathorn`, not the shouting variant).
- Do not merge on the code before it is resolved for **every** group in the pair — a cached group and a fetched group must land in the same pass (case 5).
- Do not sum ranges or keep the first group's `unit_count`; case 3 fails immediately. The unit dedupe here is not `stats.duplicatesDropped`: that counter belongs to the search layer and must not move.
- Reviewers: `grep -n "buildings\[:\]"` shows the in-place replacement; `grep -n "display_name"` shows the import, not a local loop.

## Plan samples are untrusted

The plan's rules are prose; its JSON sample shows `id: "p00348"` and `coordSource: "detail:CX-84121"` as an illustration. Treat them as the shape, not as values any test must copy; record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
