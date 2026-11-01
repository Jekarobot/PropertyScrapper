---
id: 03-11
stage: 03
title: Assemble the dataset and its stats
points: 2
owner: pipeline
status: todo
depends_on: [03-06, 03-10]
files:
  - pipeline/src/connex_watch/build_dataset.py
  - pipeline/tests/test_build_dataset.py
---

# 03-11 · Assemble the dataset and its stats

**Points:** 2 · **Stage:** 03 — Dataset builder · **Owner lane:** pipeline · **Depends on:** 03-06, 03-10 · **Unblocks:** 03-12, 03-13, 03-15

## Goal

Add `build_dataset(run_dir, *, now=None)` returning the pydantic `Dataset` of `plan/02-architecture.md` §3.3 — schema version, source, mirrored search spec, statistics, buildings with null coordinates — with an injectable clock so output is byte-stable.

## Why this task exists

This is the artefact the whole project exists to produce: one self-describing JSON file that the static web app loads. Everything before it produces or cleans raw data; everything after it checks, publishes or consumes it. Two properties make it trustworthy: the `search` block tells a user which query the numbers belong to (so a stale or wrong-preset dataset is obvious), and `generatedAt` from an injected clock plus deterministic ordering makes two runs byte-identical, which is what P8 and the golden snapshot depend on. `lat`/`lng`/`coordSource` are part of the schema from day one as `null`, so stage 04 only fills values and stage 05 can build against this dataset immediately.

## Current state (preconditions)

- 03-05…03-10 are done: `load_run`, `filter_available`, `select_units` (with `EmptyDatasetError`), `group_units`, `assign_building_ids`, `vote_location`, `sort_groups`, `sort_units` all exist and are tested.
- `pydantic` v2 is installed (00-03); `models.py` has `SearchSpec` (01-03).
- `build_dataset.py` has no dataset models and no `build_dataset` yet.

## Read before you start

- `CLAUDE.md` §3 (determinism: inject the clock), §4 (2 points: 6–12 cases, one integration case), §6.
- `plan/02-architecture.md` §3.3 — the full published schema, every field and the ordering rules; §3.1; §6 P7, P8.
- `plan/stage-03-dataset-builder.md` Task 3.4 items 1–2 and Traps (coordinates must exist as null).
- `plan/90-test-matrix.md` row P-17 — deterministic output with a frozen clock.
- `plan/stage-04-enrichment.md` — `build --no-coords` expects `buildingsWithCoords == 0`, i.e. this dataset.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/build_dataset.py` | `Source`, `DatasetStats`, `Unit`, `Building`, `Dataset`, `build_dataset` |
| create | `pipeline/tests/test_build_dataset.py` | the seven cases below |

Nothing else may change: no CLI (03-13), no validator (03-12), no `models.py` edit, no fixture edit.

## Test first (red)

1. `test_dataset_top_level_shape` — `build_dataset(run-mini, now=…)` has `schemaVersion == 1`, `generatedAt` `2026-11-01T21:45:00Z` for `now = 2026-11-01T21:45:00+00:00`, `source == {"site": "connex.in.th", "kind": "rent-condo"}`, and `search` equal to the manifest spec (ten keys, FULL values).
2. `test_dataset_stats_for_run_mini` — `units == 5`, `buildings == 3`, `buildingsWithCoords == 0`, `pagesScanned == 3`, `duplicatesDropped == 1`, `excluded == {"check_availability": 1, "not_available": 2}`, `partial is False`.
3. `test_buildings_carry_ids_ranges_location_and_units` — building order `A Tower`, `A, B Condo`, `B Place`; A Tower has `id a-tower`, `unitCount 3`, `priceMin 45000`, `priceMax 50000`, `sizeMin 58`, `sizeMax 65`, `district Sa Thon`, `province Bangkok`, and units in order `CX-1`, `CX-4`, `CX-2`.
4. `test_coordinates_are_null_and_stage_04_is_additive` — every building has `lat is None`, `lng is None`, `coordSource is None`; the keys exist in `model_dump()` and in the serialised JSON.
5. `test_json_round_trip_equality` — `Dataset.model_validate_json(ds.model_dump_json()) == ds`, and `json.loads(ds.model_dump_json())["generatedAt"] == "2026-11-01T21:45:00Z"`.
6. `test_deterministic_bytes_with_frozen_clock` — two builds with the same `now` give identical JSON strings; a build with `now` one hour later differs only in `generatedAt` (compare dicts after removing that key).
7. `test_no_excluded_unit_reaches_the_dataset` — no `CX-6`, `CX-7`, `CX-8` anywhere; every unit url starts `https://connex.in.th/`; every unit has non-null `priceThb` and `sizeSqm`; `stats.units == sum(unitCount)`.

Expected red: `ImportError: cannot import name 'build_dataset'` / `Dataset`; case 6 failing on a differing `generatedAt` if the clock is not injected.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_build_dataset.py -q`

## Implement (green)

`build_dataset(run_dir: Path, *, now: datetime | None = None) -> Dataset` composes the tested pieces in one readable pipeline: `load_run` → `select_units` (raises `EmptyDatasetError`) → `group_units` → `assign_building_ids` → `vote_location` per group → `sort_groups` / `sort_units` → models. `now` defaults to `datetime.now(timezone.utc)`; a naive value raises `ValueError` (an unaware timestamp in a published file is a bug).

Models (pydantic v2, snake_case attributes, camelCase aliases for JSON so the file matches §3.3 exactly): `Source`, `DatasetStats` (`units`, `buildings`, `buildingsWithCoords`, `pagesScanned`, `duplicatesDropped`, `excluded`, `partial`), `Unit` (`id`, `url`, `priceThb`, `sizeSqm`, `bedrooms`, `bathrooms`, `floor`, `updatedOn`), `Building` (`id`, `name`, `district`, `province`, `lat`, `lng`, `coordSource`, `unitCount`, `priceMin`, `priceMax`, `sizeMin`, `sizeMax`, `units`), `Dataset` (`schemaVersion`, `generatedAt`, `source`, `search`, `stats`, `buildings`). `generatedAt` serialises as UTC with a trailing `Z`.

`stats.partial` is the manifest's explicit `partial` flag (written by 02-12; `finish()` takes it as an argument), or `errorCount > 0`, or any page file was skipped (`RawRun.skipped_pages`). Do not re-derive it from `stopReason`: an `empty_streak` stop is the normal end of a nightly walk and is not partial. run-mini is `False`. `buildingsWithCoords` is `0` at this stage and is recomputed by stage 04. Do not add fields beyond §3.3.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_build_dataset.py -q` — seven green.

## Refactor / keep it DRY

One assembly function; per-building construction in one small helper so the ranges come from the group and the location from the vote in a single place. `stats.units` is `sum(b.unitCount)` computed once, not from `len(cards)`. Do not duplicate the ordering logic — call 03-10's sorters. Do not re-derive the search dict: pass the manifest spec through the existing model.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_build_dataset.py -q
pytest pipeline -q
ruff check pipeline
```

Manual check: a short ad-hoc call prints `model_dump_json(indent=2)` with three buildings, five units and `"lat": null`; two calls with the same `now` are byte-identical.

## Out of scope (do not do)

- No CLI, no file writing, no atomic replace — 03-13 writes the file.
- No validation rules — 03-12 reads this model.
- No enrichment, coordinates, detail pages or network (`build_dataset.py` must not import `httpx` or `fetch`).
- No extra schema fields (no `droppedInvalid`, no per-page data, no run id) beyond §3.3.
- No `datetime.now()` call outside the `now` default.

## Definition of done

- [ ] All seven cases exist, named as written, and pass; red run quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `build_dataset.py` and the new `test_build_dataset.py`.
- [ ] The output JSON matches §3.3 key-for-key (checked in the round-trip case).
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
03-11: assemble the dataset and its stats

- points: 2
- red: test_dataset_top_level_shape failed (ImportError: build_dataset)
- add Dataset/DatasetStats/Building/Unit models and build_dataset with an injected clock
- emit null lat/lng/coordSource so stage 04 stays additive
- suite: N passed
```

Files in the commit: `build_dataset.py`, `test_build_dataset.py` and this task file. Nothing else.

## Traps and review notes

- A naive `generatedAt` or `datetime.now()` inside the pipeline body makes case 6 flaky and the file non-diffable.
- pydantic v2 emits `+00:00`, not `Z`; case 5 pins the `Z`, so add the serialiser rather than weakening the test.
- `excluded` must come from `select_units`, not from a recount that also counts excluded cards of the last page twice.
- `stats.units` must equal `sum(unitCount)`; computing it from cards hides a grouping bug.
- Reviewers: `grep -rn "httpx\|fetch" pipeline/src/connex_watch/build_dataset.py` must be empty, and `model_dump()` keys must equal §3.3.

## Plan samples are untrusted

The plan's dataset JSON is illustrative: its numbers come from a full run, not run-mini, and it shows a `project_code`-based id that only enrichment produces. Take the **shape** from it and the values from this task; record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
