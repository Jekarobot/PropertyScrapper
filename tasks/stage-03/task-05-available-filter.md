---
id: 03-05
stage: 03
title: Keep only available units
points: 1
owner: pipeline
status: todo
depends_on: [03-01, 03-02]
files:
  - pipeline/src/connex_watch/build_dataset.py
  - pipeline/tests/test_build_dedupe.py
---

# 03-05 · Keep only available units

**Points:** 1 · **Stage:** 03 — Dataset builder · **Owner lane:** pipeline · **Depends on:** 03-01, 03-02 · **Unblocks:** 03-06, 03-07

## Goal

Add `load_run` and `filter_available` so the builder reads a raw run from disk and keeps exactly the `available` cards, counting the two excluded states.

## Why this task exists

The published dataset must never contain a unit that is not available — that is rule P5 and the project's central correctness claim (`plan/01-discovery.md` §5: the site cannot filter by availability, so we must). This is also the first code that opens a raw run directory, which makes it the seam between stage 02's output and stage 03's logic: if the reader is wrong, every later number is wrong. The excluded counts are the early-warning signal that the site's ordering or state mix changed between runs.

## Current state (preconditions)

- 03-01 is done: `pipeline/tests/fixtures/run-mini/` holds 3 pages (9 cards) and a manifest.
- 03-02 is done: `normalize.py` exists (not needed by this task yet, but the module layering assumes it).
- `pipeline/src/connex_watch/build_dataset.py` does **not** exist.

## Read before you start

- `CLAUDE.md` §3 (TDD), §5 (YAGNI), §6, §10 (no network in tests).
- `plan/02-architecture.md` §3.1 (the enum, "only `available` is ever published"), §3.2 (page and manifest fields), §6 P5.
- `plan/stage-03-dataset-builder.md` Task 3.2 item 1 and Traps (excluded counts are a lower bound).
- `plan/90-test-matrix.md` row P-14 — the requirement this task satisfies.
- `pipeline/tests/fixtures/run-mini/` — the 9 cards and which 6 are available.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `pipeline/src/connex_watch/build_dataset.py` | `RawRun`, `UnitSelection`, `load_run`, `filter_available`, `select_units` |
| create | `pipeline/tests/test_build_dedupe.py` | the six cases below |

Nothing else may change: no grouping, no dedupe, no statistics, no CLI.

## Test first (red)

1. `test_load_run_reads_three_pages_in_order` — `load_run(run-mini)` gives `pages_scanned == 3`, page numbers `[1, 2, 3]`, nine cards total, `skipped_pages == []`, and `spec` equal to the manifest `spec` (FULL spec values).
2. `test_only_available_cards_survive` — the available ids in order are `[CX-1, CX-2, CX-3, CX-4, CX-5, CX-1]`; `CX-6`, `CX-7` and `CX-8` appear in no result at this layer.
3. `test_excluded_counts_are_exact` — `excluded == {"check_availability": 1, "not_available": 2}` for run-mini, and both keys are present with `0` for a page whose cards are all available (never a missing key).
4. `test_filter_preserves_page_then_position_order` — the returned list is page-major and position-ascending, not sorted by price or id (`plan/02-architecture.md` §3.3 ordering happens later, in 03-10).
5. `test_filter_does_not_merge_same_name_units` — two cards of the same building (CX-1 and CX-4, `A Tower` / `a tower`) both survive: six cards in, six out; merging is 03-08's job and fuzzy merging is forbidden entirely.
6. `test_filter_of_unavailable_only_page_is_empty` — `filter_available` on page 3's three cards yields `[]` with `excluded == {"check_availability": 1, "not_available": 2}`; the pure filter never raises (03-07 guards `select_units`).

Expected red: `ModuleNotFoundError: No module named 'connex_watch.build_dataset'`, then `ImportError` for the names. A card-model validation error means 03-01's fixture and 01-02's model disagree — stop and report.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_build_dedupe.py -q`

## Implement (green)

- `load_run(run_dir: Path) -> RawRun` — read `manifest.json`, then every `pages/page-*.json` in ascending filename order, each through the 01-02 `PageResult` model. `RawRun` carries the resolved dir, the manifest values, `spec`, the pages, `pages_scanned` (files actually read) and `skipped_pages` (empty until 03-15). Missing manifest or unreadable JSON propagates the natural exception; do not swallow it here.
- `filter_available(cards) -> tuple[list[UnitCard], dict[str, int]]` — partition on `UnitState.available`, preserve input order, and return counts keyed by the two non-available enum **values** with both keys always present.
- `filter_available` and a thin `select_units(run) -> UnitSelection` (`units`, `excluded`) are the only public names. Work on `UnitCard` throughout; the smaller dataset `Unit` model is built in 03-11.
- One-sentence docstrings; no logging yet (the invalid-drop warning belongs to 03-06).

Run: `. .venv/bin/activate && pytest pipeline/tests/test_build_dedupe.py -q` — six green.

## Refactor / keep it DRY

`filter_available` is the only place that compares states to `available` — 03-11's statistics read `excluded` from `select_units`, never recount. No separate "is available" predicate in two files. Reading pages lives in `load_run` only; later tasks extend that function rather than opening `page-*.json` again.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_build_dedupe.py -q
pytest pipeline -q
ruff check pipeline
```

Manual check: a short ad-hoc call prints six selected cards and `{"check_availability": 1, "not_available": 2}`; `grep -rn "httpx" pipeline/src/connex_watch/build_dataset.py` finds nothing.

## Out of scope (do not do)

- No dedupe, no invalid-unit dropping, no `EmptyDatasetError` — 03-06 and 03-07 add them to this file.
- No grouping, statistics, sorting, dataset model or CLI.
- No network, no `httpx`, no fetcher import: the builder reads files only.
- No fuzzy or normalised-name merging in the filter.
- No writing to `data/` from this module.

## Definition of done

- [ ] All six cases exist, named as written, and pass; red run quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only the two new files.
- [ ] Every public name has a one-sentence docstring; no `httpx` import in the module.
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
03-05: keep only available units

- points: 1
- red: test_load_run_reads_three_pages_in_order failed (ModuleNotFoundError)
- add load_run: manifest plus pages in filename order, validated by PageResult
- add filter_available: available only, excluded counts keyed by state value
- suite: N passed
```

Files in the commit: `build_dataset.py`, `test_build_dedupe.py` and this task file. Nothing else.

## Traps and review notes

- Counting states from `stateCounts` instead of from the cards hides a parser bug; count the cards.
- A missing key when a state count is zero breaks the dataset schema — always emit both keys.
- Do not sort here "to be safe": 03-10 owns ordering and a premature sort changes the first-wins rule of 03-06.
- Do not convert cards to `Unit` yet: grouping needs `buildingName`, `district` and `province`.
- Reviewers: confirm the excluded dict values are the enum **strings** (`check_availability`, `not_available`), not member names.

## Plan samples are untrusted

The plan gives the four helper signatures as a sketch and no bodies. Verify the 01-02 model field names against the repo before using them; record any deviation under `## Decisions`. Do not add a dependency because a sample mentions one.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
