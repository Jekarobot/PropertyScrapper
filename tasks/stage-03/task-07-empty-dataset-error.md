---
id: 03-07
stage: 03
title: Refuse to build an empty dataset
points: 0.5
owner: pipeline
status: todo
depends_on: [03-05]
files:
  - pipeline/src/connex_watch/build_dataset.py
  - pipeline/tests/test_build_dedupe.py
---

# 03-07 · Refuse to build an empty dataset

**Points:** 0.5 · **Stage:** 03 — Dataset builder · **Owner lane:** pipeline · **Depends on:** 03-05 · **Unblocks:** 03-11

## Goal

Add `EmptyDatasetError` and make `select_units` raise it when zero units survive, so a broken site layout can never publish an empty file.

## Why this task exists

If the site changes a card selector, every page parses "successfully" with zero available units, and the pipeline would happily publish a dataset with no buildings. The web app would then show an empty map that looks like "no listings match", not like a broken crawl — the worst failure mode this project has (`plan/90-test-matrix.md` anti-pattern row "Empty dataset published after a site change"). A loud, typed error stops the nightly run before the publish step and names the run directory, so the operator can look at the raw pages instead of guessing. Validation (03-12) checks a dataset that already exists; this guard fires earlier, while building.

## Current state (preconditions)

- 03-05 is done: `load_run`, `filter_available`, `select_units` exist; six tests pass.
- Nothing raises when the selection is empty; the available-filter test for a page-3-only input expects `[]`.

## Read before you start

- `CLAUDE.md` §3 (TDD), §5, §9 (where a decision like "no stats field" is recorded).
- `plan/stage-03-dataset-builder.md` Task 3.2 item 4 — the empty-dataset rule.
- `plan/02-architecture.md` §6 P9 — `check` also treats 0 units / 0 buildings as a failure; §3.3 `stats.excluded`.
- `plan/90-test-matrix.md` row O-05 — the herd/empty guard, implemented as `test_validate.py::test_empty_dataset` in 03-14.
- `pipeline/tests/fixtures/run-mini/pages/page-0003.json` — three cards, none available.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/build_dataset.py` | `EmptyDatasetError` and the guard in `select_units` |
| modify | `pipeline/tests/test_build_dedupe.py` | the three cases below |

Nothing else may change: no CLI mapping (03-13), no validator change (03-12), no new module.

## Test first (red)

1. `test_empty_selection_raises_empty_dataset_error` — a run whose only page is page 3 raises `EmptyDatasetError`; the message contains `0 available units` and both excluded counts (`check_availability` 1, `not_available` 2).
2. `test_empty_dataset_error_names_the_run_directory` — the message contains the resolved path of the run directory, so the operator knows which raw run broke.
3. `test_non_empty_selection_does_not_raise` — run-mini returns a `UnitSelection` with five units and no exception; `EmptyDatasetError` is a `RuntimeError` subclass (so a CLI can catch it without catching every `Exception`).

Expected red: `ImportError: cannot import name 'EmptyDatasetError'`, and case 1 failing with `Failed: DID NOT RAISE`. If case 1 passes for the wrong reason (a `KeyError` from the empty run), fix the fixture wiring first.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_build_dedupe.py -q`

## Implement (green)

`class EmptyDatasetError(RuntimeError)` with a message built in one place: "0 available units in <run_dir> (excluded: check_availability=1, not_available=2)" plus the page count scanned. `select_units` raises it after dedupe and invalid drops when the list is empty — the guard sits at the end of the selection, not in `load_run`, because an empty *selection* is the publishable-empty condition while an empty *run directory* is an IO error. One sentence of docstring on the exception. No logging before raising (the CLI prints the message once).

Run: `. .venv/bin/activate && pytest pipeline/tests/test_build_dedupe.py -q` — fifteen green.

## Refactor / keep it DRY

The message is composed once, from the counts `select_units` already has; do not re-count states to build it. Keep the exception in `build_dataset.py` so 03-13 imports it from the module that raises it — no separate `errors.py` for one class.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_build_dedupe.py -q
pytest pipeline -q
ruff check pipeline
```

Manual check: a run pointed at a directory holding only page 3 raises with the path and the two excluded counts visible on one line.

## Out of scope (do not do)

- No CLI exit code mapping — 03-13 maps this error to its exit code.
- No validation rule and no `test_validate.py` change — 03-12/03-14 handle an existing dataset.
- No fallback dataset, no "keep yesterday's file" logic, no publish logic.
- No new stats field recording emptiness.
- No retry, no second read of the raw directory.

## Definition of done

- [ ] All three cases exist, named as written, and pass; red run quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `build_dataset.py` and `test_build_dedupe.py`.
- [ ] The exception message contains the run path and both excluded counts.
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
03-07: refuse to build an empty dataset

- points: 0.5
- red: test_empty_selection_raises_empty_dataset_error failed (ImportError: EmptyDatasetError)
- add EmptyDatasetError raised by select_units when no unit survives
- name the run directory and the excluded counts in the message
- suite: N passed
```

Files in the commit: `build_dataset.py`, `test_build_dedupe.py` and this task file. Nothing else.

## Traps and review notes

- Raising inside `filter_available` would break 03-05's legitimate "page 3 alone is empty" test; raise at the end of `select_units` only.
- Catching `Exception` in the CLI later would also swallow parse bugs; the error must be its own type — assert the `RuntimeError` subclass relationship.
- An empty run directory still raises the IO exception, not this one; do not merge the two conditions.
- Reviewers: confirm the guard runs **after** invalid drops, so a run of only-invalid cards also fails loudly.

## Plan samples are untrusted

The plan names `EmptyDatasetError` without a base class or message contract. The base class and message contents above are this task's choice; follow them and record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
