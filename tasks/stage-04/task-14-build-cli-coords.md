---
id: 04-14
stage: 04
title: build CLI with coordinate flags and dry-run
points: 2
owner: pipeline
status: todo
depends_on: [04-09, 03-13]
files:
  - pipeline/src/connex_watch/cli.py
  - pipeline/tests/test_cli_build.py
---

# 04-14 · build CLI with coordinate flags and dry-run

**Points:** 2 · **Stage:** 04 — Enrichment · **Owner lane:** pipeline · **Depends on:** 04-09, 03-13 · **Unblocks:** 04-15

## Goal

Extend `connex-watch build` with `--coords`/`--no-coords`, `--cache`, `--max-coord-fetches` and `--dry-run`, wiring the enrichment loop into the dataset build.

## Why this task exists

Stage 03's `build` is deliberately offline: raw pages in, dataset out. Stage 04 adds the one part of the pipeline that touches the network, and it must be a switch the operator holds, not a side effect of building. `--no-coords` keeps the offline rebuild reproducible and makes the whole suite network-free; `--dry-run` answers "what would this cost?" before a cold run spends twenty minutes; `--cache` is what the nightly script points at. Without this task the enrichment loop is a library nobody calls, and the dataset keeps its null coordinates forever.

## Current state (preconditions)

- 03-13 done: `connex-watch build --raw DIR --out FILE [--no-coords]` parses arguments, calls `build_dataset`, writes atomically, prints one summary line (`units=5 buildings=3 pages=3 partial=false` for `run-mini`) and exits 0/1/2/4. `--no-coords` is already accepted there and is a no-op, because no enrichment code exists yet.
- 04-09, 04-10, 04-11 done: `enrich`, the caps, `EnrichReport` and the progress log; 04-12/04-13 may be done too (merge changes the building list but not this CLI's flags).
- The CLI already builds its fetcher in `cli._make_fetcher` (02-13), which the `crawl` tests monkeypatch; reuse that one seam. `pipeline/tests/test_cli_build.py` exists with 03-13's seven cases; `pytest pipeline -q` is green.

## Read before you start

- `plan/stage-04-enrichment.md` Task 4.5 — the flag list and the five CLI cases.
- `plan/02-architecture.md` §3.3 (`stats.buildingsWithCoords`, `stats.partial`) and §6 rules P6/P7. `plan/stage-08-hardening-ops.md` Task 8.1 — the nightly build passes `--cache` and **no** coordinate flag, which is why coordinates default to on.
- `pipeline/src/connex_watch/cli.py` and `pipeline/tests/test_cli_crawl.py` — argument conventions and the fetcher seam.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/cli.py` | the four new options and the enrichment call |
| modify | `pipeline/tests/test_cli_build.py` | 03-13's cases stay; the seven new cases below are appended |

Nothing else may change: no `build_dataset.py`, no `enrich.py`, no `Makefile` target, no new dependency.

## Test first (red)

Run the CLI through its `main(argv)` entry point with `tmp_path` outputs and `cli._make_fetcher` monkeypatched to a fake; every recorded URL is asserted, so a stray request fails the test.

1. `test_cli_build.py::test_no_coords_is_offline_and_zero_coords` — `--raw` `run-mini`, `--out` tmp, `--no-coords` → exit 0, `stats.buildingsWithCoords == 0`, `stats.buildings == 3`, every building `lat is None`, and the fetcher seam was never called.
2. `...::test_coords_dry_run_reports_cache_misses_without_requests` — cold `--cache`, `--coords --dry-run` → output states 3 buildings would be fetched, zero requests, **no** dataset file written, exit 0.
3. `...::test_coords_with_fake_fetcher_sets_buildings_with_coords` — `--coords`, fake returns the detail fixture per URL with a distinct project code so nothing merges → exit 0, `stats.buildingsWithCoords == 3`, `stats.units == 5`, and the cache file holds three entries.
4. `...::test_cache_flag_writes_to_the_given_path` — `--cache tmp/buildings.json` → that file exists with three entries and the default `data/cache/buildings.json` was not created.
5. `...::test_coords_are_enabled_by_default` — no coordinate flag at all (only `--cache`, as the nightly script does) → three requests and `stats.buildingsWithCoords == 3`.
6. `...::test_max_coord_fetches_is_passed_through` — `--coords --max-coord-fetches 1` → exactly one request, `stats.buildingsWithCoords == 1`, exit 0.
7. `...::test_no_search_page_is_refetched` — every recorded URL is a detail URL; none contains `search_en`, and no URL appears twice.

Expected red: `SystemExit: 2` with `unrecognized arguments: --coords` (argparse) for cases 2–7 — none of those flags exists yet. Case 1 is already green from 03-13 (the `--no-coords` no-op) and must stay green; if it breaks, the new default leaked into the old path.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_cli_build.py -q`

## Implement (green)

- Options on `build`: `--coords` / `--no-coords` (mutually exclusive, default **on** — the nightly command passes `--cache` and no flag, and a nightly build without coordinates is pointless), `--cache PATH` (default `data/cache/buildings.json`), `--max-coord-fetches N` (default 1200), `--dry-run`.
- `--no-coords`: call `build_dataset`, set `stats.buildingsWithCoords = 0` (or leave it at the stage-03 value of 0), write, print the summary, and never construct a fetcher.
- `--coords`: load the cache (`CoordCache.load`), run `enrich(buildings, fetcher, cache, max_fetches=…, now=…)`, save the cache after the loop, then write the dataset with `stats.buildingsWithCoords` = buildings with non-null coordinates (that is the JSON key; use the attribute name 03-11 gave `DatasetStats`, never a second stat). The run's `now` is taken once and passed to both `enrich` and the dataset stamping so the output stays reproducible under a frozen clock.
- `--dry-run`: print the number of cache misses for the candidate set (use `cache.pending`/the candidate rule, not a second implementation) and exit 0 without writing the dataset and without a request; it must not create the cache file. Build the fetcher through the existing `cli._make_fetcher` (02-13) — no `--fetcher` flag, no test-only branch in `cli.py`.
- Keep 03-13's exit codes (0 success, 2 usage, 4 unreadable raw page, 5 empty selection) and its atomic write. One-sentence help text on each new flag; docstring line on the dry-run contract.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_cli_build.py -q` — fourteen green (the seven new ones plus 03-13's seven).

## Refactor / keep it DRY

The cache path default is one constant shared with `coord_cache.py`, not a second literal. Candidate selection for the dry-run is `enrich`'s function, not a copy. The summary line is printed once, after the dataset is written, whatever the flags.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_cli_build.py -q
. .venv/bin/activate && pytest pipeline -q
connex-watch build --raw pipeline/tests/fixtures/run-mini --out /tmp/ds.json --no-coords
ruff check pipeline
```

Expected: 14 passed, suite green, the manual build exits 0 and prints the usual `units=… buildings=… pages=… partial=false` line, `ruff` exits 0. The manual command must make no network call — run it with the venv active and a clean cache path.

## Out of scope (do not do)

- Partial/capped run reporting, warnings and the corrupt-cache exit code — 04-15.
- Re-crawling search pages, `--resume`, or any crawl flag on `build`. New Makefile targets, a `--verbose` flag, or logging configuration beyond what 03-13/02-13 already set up.
- Changing the dataset schema, the flag defaults of `crawl`, or the `check` command.

## Definition of done

- [ ] All seven cases exist, named as written, and pass; no test writes outside `tmp_path`.
- [ ] Red run observed and one failure line quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean; `--no-coords` path provably offline.
- [ ] Only the two files above plus this task file are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
04-14: wire coordinate enrichment into the build CLI

- points: 2
- red: test_coords_dry_run_reports_cache_misses_without_requests failed (SystemExit: 2 unrecognized arguments: --coords)
- add --coords/--no-coords, --cache, --max-coord-fetches and --dry-run
- enrich, save the cache and publish buildingsWithCoords
- suite: N passed
```

Files in the commit: the two files above and this task file (status + Result). Nothing else.

## Traps and review notes

- `--no-coords` must not construct a fetcher at all: "construct but never call" still reads config, and a test that only checks the call count misses it.
- `--dry-run` must not create or rewrite the cache file; a dry run that mutates state is not a dry run. Defaulting coordinates to **off** would make the nightly build publish an empty map while looking healthy — case 5 is the guard.
- The `now` timestamp must be shared by enrichment and the dataset, or two runs of the same input differ by a second and the determinism test in 03-11 starts failing intermittently. Do not print per-building progress here: `enrich` logs it (04-11).
- Reviewers: `grep -n "parse_args\|add_argument" cli.py` shows one option each; `grep -n "pending\|candidate"` in `cli.py` shows a call, not a re-implementation.

## Plan samples are untrusted

The plan shows the exact command line but no code. Verify every flag against the CLI as 03-13 built it, and check `--raw`/`--out` spellings before documenting them (`CLAUDE.md` §6). Record the default-coordinate decision under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
