---
id: 04-15
stage: 04
title: Partial enrichment and corrupt-cache exit
points: 1
owner: pipeline
status: todo
depends_on: [04-14]
files:
  - pipeline/src/connex_watch/cli.py
  - pipeline/tests/test_cli_build.py
---

# 04-15 · Partial enrichment and corrupt-cache exit

**Points:** 1 · **Stage:** 04 — Enrichment · **Owner lane:** pipeline · **Depends on:** 04-14 · **Unblocks:** 08-01

## Goal

Make the two degraded enrichments behave: a stopped run still publishes a dataset marked partial with exit 0, and an unreadable cache stops the build with exit 2 and a delete hint.

## Why this task exists

A build that dies because the site went quiet at building 400 produces nothing, while a build that publishes 400 mapped buildings and says so produces a usable map. The dataset already has `stats.partial` for exactly this, and the web status bar (F11) will show the warning, so the CLI must fill it instead of hiding the stop. The opposite case is the corrupt cache: continuing there would spend a thousand requests on a file the operator can fix in one command, so the run must refuse and say what to delete. Together these are the operator-facing half of P6 — the loop knows how to stop, this task makes the stop legible.

## Current state (preconditions)

- 04-14 done: `build` accepts `--coords/--no-coords/--cache/--max-coord-fetches/--dry-run`, calls `enrich`, saves the cache, writes `stats.buildingsWithCoords`.
- 04-10 provides `report.capped`/`report.aborted`, 04-11 the frozen report field set and the progress log; 04-07 provides `CacheCorruptError` with the path and the delete hint; 03-11 provides `stats.partial`; 03-12/03-14 provide `validate_dataset` and `check`.
- `pipeline/tests/test_cli_build.py` exists with the seven 04-14 cases. `pytest pipeline -q` is green.

## Read before you start

- `plan/stage-04-enrichment.md` Task 4.5 cases 4–5 and the acceptance criteria.
- `plan/02-architecture.md` §3.3 (`stats.partial`, `coordSource`), §6 rules P6/P9. `plan/90-test-matrix.md` row P-23 and anti-pattern "Coordinates fetched per unit".
- `plan/assets/fixtures/README.md` — the DEV query parameters of `search-dev-p1.html` (project `p00348`, `minsize=55`, `minprice=25000`, `maxprice=50000`, `sort=1`), needed to build a one-page raw run in the test.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/cli.py` | partial marking, warnings, `CacheCorruptError` exit |
| modify | `pipeline/tests/test_cli_build.py` | the five new cases |

Nothing else may change: no `enrich.py`, no `build_dataset.py`, no `check` command, no fixture edits.

## Test first (red)

1. `test_cli_build.py::test_aborted_enrichment_still_writes_the_dataset` — a fake fetcher raising `FetchError` for five consecutive buildings → exit 0, the dataset file exists, `stats.partial is True`, and one warning line says enrichment was aborted.
2. `...::test_corrupt_cache_exits_two_with_delete_hint` — `--cache` pointing at a file containing `nope` → exit 2, the error line contains the path and the "delete the file" hint, no dataset is written, and no request is made.
3. `...::test_bangkok_sathorn_group_gets_project_code_and_coord_source` — a raw run written to `tmp_path` in the `run-mini` layout (a `manifest.json` for the DEV query plus `pages/page-0001.json` serialised from 01-17's `parse_search_page(search-dev-p1.html)` as a 01-02 `PageResult`, which 03-05's `load_run` reads back) → one building, `id == "p00348"`, `name == "The Bangkok Sathorn"`, `unit_count == 4`, `lat == 13.7187616`, `lng == 100.5202356`, `coord_source == "detail:CX-135510"`, `stats.buildingsWithCoords == 1`.
4. `...::test_capped_run_warns_and_exits_zero` — `run-mini` with `--max-coord-fetches 1` → exit 0, the dataset exists with `stats.buildingsWithCoords == 1`, and a warning mentions the cap.
5. `...::test_aborted_run_still_saves_the_cache` — after the aborted run, reload the cache file: the resolved keys are there, so the next run retries only the failures.

Why `CX-135510` in case 3: the four available cards on that page cost ฿45 000 (`CX-84121`, `CX-135510`), ฿50 000 (`CX-174761`) and ฿50 000 (`CX-139586`); the cheapest is a tie and the tie-break is plain id order, where `CX-135510` sorts before `CX-84121`.

Expected red: case 1 fails with `assert False is True` on `stats.partial` (or a `FetchError` escaping as a traceback), case 2 exits 1/3 instead of 2, case 3 fails on the building id because a stage-03 slug is still in place.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_cli_build.py -q`

## Implement (green)

- After `enrich`, if `report.aborted` (or `report.capped`), log a WARNING naming which limit stopped the run and how many buildings have coordinates; keep going to the write. `enrich` already logged the per-25 progress.
- `stats.partial = True` when `report.aborted` is true — the map is published and the UI flags it. Do not touch `stats.partial` for a capped run; add a one-line comment saying why (the cap is a budget, and the owner's rule for `partial` is "aborted or a page failed").
- Exit 0 with the dataset written in both degraded cases. The warning goes to the log/stderr, never into the dataset.
- `CacheCorruptError` from `CoordCache.load` is caught at the CLI boundary only: print the message (it already contains the path and the delete hint), write nothing, make no request, exit 2. Do not catch it inside `enrich`.
- `stats.buildingsWithCoords` and `coordSource` come from the enrichment result as written in 04-14; do not recompute them differently here.
- Keep 03-13's summary keys (`units=`, `buildings=`, `pages=`, `partial=`) so the existing nightly log line stays one line and still shows the state.
- Docstring/comment: one line on the exit-2 contract for a corrupt cache.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_cli_build.py -q` — nineteen green.

## Refactor / keep it DRY

The degraded-run warning is one helper called for aborted and capped, differing only by a reason string. `stats.partial` is set in one place. The exit-code mapping stays the one 03-13/02-14 established; do not invent a new code for the cache (2 is the usage/material-error code).

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_cli_build.py -q
. .venv/bin/activate && pytest pipeline -q
connex-watch build --raw pipeline/tests/fixtures/run-mini --out /tmp/ds.json --no-coords
ruff check pipeline
```

Expected: 19 passed, suite green, the manual build exits 0 and prints `partial=false`, `ruff` exits 0. Manual corrupt-cache check: write `nope` into a temp cache path, run the same command with `--coords --cache <path>`, and confirm exit 2 plus the hint — with no request made.

## Out of scope (do not do)

- Changing `check`'s exit policy for `stats.partial` — that is P9 and stage 08's business.
- Retrying the failed buildings inside this task, or a `--retry-failed` flag. Making a capped run `partial` (see the comment above); if the owner wants it, it is a separate change with its own test.
- Writing diagnostics into the dataset, adding fields to `stats`, or editing the fixtures. Catching `BlockedError` or any non-cache exception at this boundary.

## Definition of done

- [ ] All five cases exist, named as written, and pass; the seven 04-14 cases still pass.
- [ ] Red run observed and one failure line quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean; no network in any test.
- [ ] Only the two files above plus this task file are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
04-15: publish a partial dataset when enrichment stops

- points: 1
- red: test_aborted_enrichment_still_writes_the_dataset failed (assert False is True)
- mark stats.partial on an aborted enrichment and warn, exit 0
- map CacheCorruptError to exit 2 with the delete hint
- suite: N passed
```

Files in the commit: the two files above and this task file (status + Result). Nothing else.

## Traps and review notes

- Do not swallow the abort as success: the run is green, the dataset must say `partial=True`, and the warning must name the limit.
- Do not write the dataset before the cache is saved; a crash between the two would lose the work that was already paid for.
- Case 3 is the end-to-end proof of P7 and the tie-break rule; if it lands on `CX-84121` the candidate ordering was implemented numerically or by page position.
- Do not print the html, the key, or a stack trace for the corrupt cache: one line, path plus hint. Reviewers: `grep -n "partial" cli.py` shows one assignment; `grep -n "except CacheCorruptError"` shows one handler at the boundary.

## Plan samples are untrusted

The plan states the behaviours and the flags in prose. Verify the CLI entry point, the summary keys and the exit codes against 03-13/02-14 as implemented (`CLAUDE.md` §6) before asserting them here; record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
