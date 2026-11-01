---
id: 03-09
stage: 03
title: Building ids and slug collisions
points: 0.5
owner: pipeline
status: todo
depends_on: [03-08]
files:
  - pipeline/src/connex_watch/build_dataset.py
  - pipeline/tests/test_build_group.py
---

# 03-09 · Building ids and slug collisions

**Points:** 0.5 · **Stage:** 03 — Dataset builder · **Owner lane:** pipeline · **Depends on:** 03-08 · **Unblocks:** 03-10, 03-11

## Goal

Give every group a deterministic `id` = `slugify(display_name)`, resolving slug collisions with a `-2`, `-3` suffix and a warning.

## Why this task exists

`buildings[].id` is the key the web app uses for selection, map bubbles and `key=` props, so it must be unique and stable between runs — otherwise React re-mounts rows and the selected building jumps. Until enrichment (stage 04) supplies a `project_code`, the id is derived from the display name; `plan/02-architecture.md` §3.1 says exactly that, and stage 04 later replaces it for buildings with a known project. Two different names can slug identically (`A Tower` and `A-Tower` are different normalised keys), so collisions must be resolved loudly and deterministically, not silently overwritten.

## Current state (preconditions)

- 03-08 is done: `UnitGroup` and `group_units` exist; eight group tests pass.
- `UnitGroup` has no `id` field yet.

## Read before you start

- `CLAUDE.md` §3 (TDD), §5 (no abstraction for one caller), §6.
- `plan/02-architecture.md` §3.1 (Building identity: `project_code` when known, else `slug(building_name)`), §3.3 (`buildings[].id`), §6 P8 (deterministic output).
- `plan/stage-03-dataset-builder.md` Task 3.3 item 3 — the `-2` suffix and the warning.
- `pipeline/src/connex_watch/normalize.py` — `slugify`'s Thai and 80-char behaviour, including the empty-slug case.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/build_dataset.py` | add `id` to `UnitGroup`; `assign_building_ids` |
| modify | `pipeline/tests/test_build_group.py` | the four cases below |

Nothing else may change: no ordering (03-10), no dataset model, no project-code logic (stage 04).

## Test first (red)

1. `test_building_id_is_the_slug_of_the_display_name` — A Tower → `a-tower`; `Condo, A, B` → `condo-a-b`; a Thai group `เดอะ บางกอก` → `เดอะ-บางกอก` (not transliterated).
2. `test_ids_are_stable_across_calls` — assigning ids twice to the same groups returns identical ids, and the id of a group depends only on its own display name when no collision exists.
3. `test_slug_collision_gets_numeric_suffix_and_warns` — groups named `A Tower`, `A-Tower`, `A/Tower` (three distinct keys, one slug) get `a-tower`, `a-tower-2`, `a-tower-3` in input order, with one WARNING per collision naming both names.
4. `test_empty_slug_falls_back_to_building` — a display name of `...` (slug `""`) becomes `building`, and a second such group becomes `building-2` with a warning.

Expected red: `AttributeError: 'UnitGroup' object has no attribute 'id'` and `ImportError: cannot import name 'assign_building_ids'`.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_build_group.py -q`

## Implement (green)

`assign_building_ids(groups) -> list[UnitGroup]` returns groups with `id` set (or mutates and returns the same list — pick one and say so in the docstring). Base id is `slugify(group.display_name)`; an empty base becomes `building`. Suffixes are `-2`, `-3`, … in the order groups are given, and every suffixed id logs one WARNING naming the display names that collided. A `seen` set enforces uniqueness. No `id` may be empty, contain whitespace, or start/end with `-`. One sentence of docstring.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_build_group.py -q` — twelve green.

## Refactor / keep it DRY

Collision numbering is one small helper (`_unique_id(base, seen)`) reused by the empty-slug fallback; do not duplicate the loop. `slugify` stays the only slug producer — no second `re.sub` here.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_build_group.py -q
pytest pipeline -q
ruff check pipeline
```

Manual check: run-mini's ids are `a-tower`, `b-place`, `condo-a-b` and no warning is logged (no collision).

## Out of scope (do not do)

- No project-code ids and no enrichment — stage 04 rewrites the id for buildings with a known `project_code`.
- No hashing, uuid, random suffix or sequence numbering across runs.
- No reordering of groups (03-10) and no dataset assembly (03-11).
- No change to `slugify`'s contract or to its existing tests.
- No silent overwrite of a duplicate id.

## Definition of done

- [ ] All four cases exist, named as written, and pass; red run quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `build_dataset.py` and `test_build_group.py`.
- [ ] `assign_building_ids` has a one-sentence docstring; ids are unique in every listed case.
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
03-09: assign building ids with slug collisions

- points: 0.5
- red: test_building_id_is_the_slug_of_the_display_name failed (AttributeError: id)
- add id = slugify(display_name) with -2/-3 collision suffixes and a warning
- fall back to "building" for an empty slug
- suite: N passed
```

Files in the commit: `build_dataset.py`, `test_build_group.py` and this task file. Nothing else.

## Traps and review notes

- Do not make the suffix depend on a global counter or the wall clock: the same input must always give the same ids (P8).
- Suffix order is the input order of groups, which 03-10 makes deterministic by sorting afterwards; do not sort inside this function.
- A Thai slug must stay Thai; a suffix on a Thai id is `เดอะ-บางกอก-2`.
- Reviewers: confirm `seen` is checked **after** suffixing (a loop that returns `base` twice passes case 1 and fails case 3).

## Plan samples are untrusted

The plan says "collisions handled by the caller (Stage 04)" in one place and "collision gets `-2` suffix and a warning" in another. This task implements the suffix now, before enrichment; record the contradiction under `## Decisions` when implementing.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
