---
id: 04-13
stage: 04
title: Same-name projects stay separate
points: 0.5
owner: pipeline
status: todo
depends_on: [04-12]
files:
  - pipeline/src/connex_watch/enrich.py
  - pipeline/tests/test_enrich_merge.py
---

# 04-13 · Same-name projects stay separate

**Points:** 0.5 · **Stage:** 04 — Enrichment · **Owner lane:** pipeline · **Depends on:** 04-12 · **Unblocks:** nothing yet

## Goal

Harden the merge: a shared name with different project codes is **not** a merge, codeless groups keep their suffixed ids, and disagreeing coordinates are resolved by rule with a warning.

## Why this task exists

The merge key is the project code, not the title — that is the whole point of 04-12, and it is exactly what breaks when someone "simplifies" the merge back to names. Two genuinely different projects can share a display name (the site's titles are free text, and stage 03's slug collision already shows two groups fighting for one slug). Merging them would silently delete units from the dataset: one building's listing would vanish behind another's. The coordinate corner matters for the same reason — a merge must never lose a position, and when two sources disagree a human needs to see it in the log.

## Current state (preconditions)

- 04-12 done: merge by `project_code`, in-place list replacement, `display_name`/range reuse, `report.merged_groups`; seven merge cases green.
- `enrich.py` currently merges on the code and says nothing when two groups share a name but differ in code or coordinates.
- `pytest pipeline -q` is green.

## Read before you start

- `plan/stage-04-enrichment.md` Task 4.4 cases 3–4 — the collision and the coordinate disagreement.
- `plan/02-architecture.md` §6 rule P7 (merge on `project_code`) and §3.1 (`Building` identity).
- `pipeline/src/connex_watch/enrich.py` — the grouping map 04-12 introduced.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/enrich.py` | code-mismatch guard, suffix preservation, coordinate rule, warnings |
| modify | `pipeline/tests/test_enrich_merge.py` | the four new cases |

Nothing else may change: no cache change, no CLI, no logging configuration.

## Test first (red)

1. `test_enrich_merge.py::test_different_project_codes_with_the_same_name_are_not_merged` — two groups with the same display name resolving to `p00348` and `p00411` → two buildings, no unit moved between them, and a warning on the `enrich` logger.
2. `...::test_codeless_groups_with_colliding_slugs_keep_their_suffix_ids` — two groups whose pages yield no project code, arriving with the stage-03 ids `a-tower` and `a-tower-2` → both survive untouched, a warning is logged.
3. `...::test_merge_keeps_the_first_non_null_coordinates` — two entries both claiming `p00348`, one with `13.7187616` / `100.5202356` and one with null coordinates → the merged building carries the non-null pair and its `coord_source`.
4. `...::test_disagreeing_coordinates_log_a_warning` — both entries non-null but different → the first (in building order) wins and the warning names both coordinate pairs.

Expected red: `AssertionError: assert 1 == 2` on the building count for cases 1–2 (the current merge would fold them, or would drop the suffix), and an empty `caplog` for the warning assertions.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_enrich_merge.py -q`

## Implement (green)

- Merge key stays the project code. Two groups with the same name and different non-null codes are two buildings: log a warning naming the shared name and both codes, and keep both (`id` = each group's own code).
- Groups with no code are never merged into anything; if two of them collided in stage 03 the suffixed id from 03-09 is preserved verbatim, with a warning.
- Coordinate rule on merge: first non-null pair in input order wins, together with its `coord_source`; if another contributing group has a non-null, different pair, log one warning containing both pairs (no averaging, no "latest wins").
- Warnings use the module logger at WARNING level, `logging.getLogger(__name__)`; no `print`, no exception — a collision is data, not a crash.
- Do not change `merged_groups`: a rejected merge is not a merge and must not be counted.
- Docstring: one sentence stating the code is the only merge key and that name collisions warn.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_enrich_merge.py -q` — eleven green.

## Refactor / keep it DRY

The "can these two merge?" predicate lives in one place used by the group scan; the coordinate choice is one helper shared with 04-12's single-group path. Do not duplicate the warning formatting in two branches — one message builder, two call sites if needed.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_enrich_merge.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 11 passed, whole suite green, `ruff` exits 0. Manual check: `grep -n "logger.warning" pipeline/src/connex_watch/enrich.py` shows the collision and the disagreement messages only.

## Out of scope (do not do)

- Fuzzy matching, case-insensitive merging of different codes, or a manual override map.
- Raising on a collision: the run must continue and publish what it has.
- Re-slugging, re-numbering ids, or "fixing" stage-03's `-2` suffix.
- Changing `report.merged_groups` semantics or the CLI.

## Definition of done

- [ ] All four cases exist, named as written, and pass; the seven 04-12 cases still pass.
- [ ] Red run observed and one failure line quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] Only the two files above plus this task file are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
04-13: keep same-name different-project buildings apart

- points: 0.5
- red: test_different_project_codes_with_the_same_name_are_not_merged failed (assert 1 == 2)
- merge only on an equal project code and warn on name collisions
- preserve suffixed ids and the first non-null coordinate pair
- suite: N passed
```

Files in the commit: the two files above and this task file (status + Result). Nothing else.

## Traps and review notes

- The dangerous shortcut is "same name, so merge": it deletes a real building's units and is invisible in the count. Case 1 is the guard.
- Do not merge a codeless group into a coded one with the same name — the code is unknown, not equal.
- Averaging or last-wins coordinates hides a source disagreement; the first non-null with a warning is the rule.
- Do not raise, and do not count a rejected collision in `merged_groups`.
- Reviewers: one warning call site per condition, no `except`, no `title.lower()` comparison anywhere in the merge.

## Plan samples are untrusted

The plan states the collision rule and the coordinate rule in bullets only. Keep the behaviour and the warning level exactly as described; record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
