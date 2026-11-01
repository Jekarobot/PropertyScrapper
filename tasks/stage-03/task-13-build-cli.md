---
id: 03-13
stage: 03
title: build subcommand with atomic write
points: 1
owner: pipeline
status: todo
depends_on: [03-11]
files:
  - pipeline/src/connex_watch/cli.py
  - pipeline/tests/test_cli_build.py
---

# 03-13 · build subcommand with atomic write

**Points:** 1 · **Stage:** 03 — Dataset builder · **Owner lane:** pipeline · **Depends on:** 03-11 · **Unblocks:** 04-14, 08-02

## Goal

Add `connex-watch build --raw DIR --out FILE [--no-coords]`: build the dataset, write it atomically, print a one-line summary, and exit 0 / 2 / 4 / 5 for success, usage, bad raw data and empty selection.

## Why this task exists

The dataset must reach `data/runs/<date>/dataset.json` and later `web/public/data/dataset.json` in one step, or a crash mid-write leaves a truncated file that the web app happily loads. `plan/02-architecture.md` §3.4 requires atomic publish; the nightly script (stage 08) and enrichment (04-14) call this command, so its exit codes and its single summary line are the contract the operator and the tests rely on. `--no-coords` exists from the start so stage 04 can add `--coords` without changing the command line operators already use; today it is accepted and is simply the only behaviour (no network at all).

## Current state (preconditions)

- 03-11 is done: `build_dataset(run_dir, *, now=None)` returns a `Dataset`.
- 03-07 is done: `EmptyDatasetError` exists in `build_dataset.py`.
- `pipeline/src/connex_watch/cli.py` exists from 02-13 with the `crawl` subcommand and its exit codes 0/2/3/4; `test_cli_build.py` does not exist.

## Read before you start

- `CLAUDE.md` §3 (TDD), §4 (1 point: 3–8 cases), §6 item 6 (run `--help` before documenting a flag), §10.
- `plan/02-architecture.md` §3.4 (one file, atomic publish), §6 P5–P9; §7 (`build` < 2 s).
- `plan/stage-03-dataset-builder.md` Task 3.4 item 4 — the printed summary; Acceptance criteria.
- `plan/stage-04-enrichment.md` Task 4.14 — `--coords/--no-coords/--dry-run` extend this command.
- `pipeline/src/connex_watch/cli.py` — how 02-13/02-14 structure `main(argv)`, argument parsing and exit codes.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/cli.py` | the `build` subparser and its handler only |
| create | `pipeline/tests/test_cli_build.py` | the seven cases below |

Nothing else may change: no `crawl` behaviour, no `check` subcommand (03-14), no `build_dataset.py` edit.

## Test first (red)

1. `test_build_cli_writes_the_dataset_and_prints_a_summary` — `main(["build", "--raw", run-mini, "--out", tmp/"ds.json"])` exits 0, the file parses as JSON, `validate_dataset` on it returns `[]`, and stdout is exactly one line `units=5 buildings=3 pages=3 partial=false`.
2. `test_build_cli_no_coords_flag_is_accepted` — with `--no-coords` the exit code, summary and JSON are identical to case 1 except `generatedAt`, and `buildingsWithCoords` is `0` for both.
3. `test_build_cli_write_is_atomic` — after a successful run no `*.tmp` file remains beside the output, and an existing `ds.json` is replaced with the new content (assert the mtime or the content changes, not that the file was created).
4. `test_build_cli_creates_the_output_directory` — `--out tmp/a/b/ds.json` where `a/b` does not exist exits 0 and creates both directories.
5. `test_build_cli_empty_selection_exits_1_and_writes_nothing` — a raw dir whose only page is page 3 (all cards unavailable) exits 1, prints one line containing `EmptyDatasetError` and `0 available units`, and leaves no output file.
6. `test_build_cli_missing_raw_directory_exits_2` — a nonexistent `--raw` path exits 2 with a message naming the path, and writes no file.
7. `test_build_cli_corrupt_page_exits_4` — a page file containing `not json` exits 4, names the file, and leaves no output file (a partial dataset is never written).

Expected red: `SystemExit` code 2 for every case (`invalid choice: 'build'`) — the subcommand does not exist yet. A `ValidationError` traceback instead of an exit code means the exception mapping is missing.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_cli_build.py -q`

## Implement (green)

Subparser `build` with `--raw` (required, directory), `--out` (required, file), `--no-coords` (store-true; today a no-op that is documented as "the only behaviour until stage 04"). Handler: create the output directory, call `build_dataset(raw)`, serialise once with the model's JSON, write `out.with_suffix(out.suffix + ".tmp")`, `os.replace` onto the target, then print exactly `units=… buildings=… pages=… partial=…` using `stats` and lower-case `true`/`false`. Catch `EmptyDatasetError` → exit 1 (the plan's data-failure code); `JSONDecodeError`/`ValidationError` from a page → exit 4 naming the file; `FileNotFoundError`/`NotADirectoryError` on `--raw` → exit 2. No traceback on stdout for these three; no network imports.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_cli_build.py -q` — seven green.

## Refactor / keep it DRY

One `_write_atomically(path, text)` helper (if 02-10 already has one in `store.py`, import it instead of copying — check first). One summary-formatting function shared later with `check`'s success line. Exception-to-exit-code mapping lives in one place in `cli.py`, not in three `try` blocks.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_cli_build.py -q
pytest pipeline -q
connex-watch build --raw pipeline/tests/fixtures/run-mini --out /tmp/ds.json
ruff check pipeline
```

Manual check: the command prints `units=5 buildings=3 pages=3 partial=false`; `/tmp/ds.json` exists, `python -m json.tool` accepts it, and no `/tmp/ds.json.tmp` remains.

## Out of scope (do not do)

- No `check` subcommand (03-14) and no `--coords`, `--dry-run`, cache or enrichment (04-14).
- No network, no `httpx`, no fetcher: `build` is offline by definition.
- No publish to `web/public/data/`, no retention, no logging framework change.
- No new dependency and no change to `crawl`'s flags, messages or exit codes.
- No `--now` flag: determinism is tested at the `build_dataset` level.

## Definition of done

- [ ] All seven cases exist, named as written, and pass; red run quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `cli.py` and the new `test_cli_build.py`.
- [ ] `connex-watch build --help` shows `--raw`, `--out`, `--no-coords` (checked manually).
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
03-13: add build subcommand with atomic write

- points: 1
- red: test_build_cli_writes_the_dataset_and_prints_a_summary failed (SystemExit 2, invalid choice)
- add build --raw/--out/--no-coords writing via .tmp plus os.replace
- map EmptyDatasetError to 1, corpus errors to 4, missing raw dir to 2
- suite: N passed
```

Files in the commit: `cli.py`, `test_cli_build.py` and this task file. Nothing else.

## Traps and review notes

- Writing straight to `--out` passes every happy-path test and fails case 3's atomicity check.
- Exit 5 must not be reused for IO errors; 0/2/3/4 already mean something in `crawl` (02-14), so pick the free code and document it.
- Printing a traceback on a corrupt page breaks the one-line summary contract case 1 asserts.
- `--no-coords` must not print a deprecation warning: stage 04's tests run it as a normal flag.
- Reviewers: grep for `open(out, "w")` on the final path and for `datetime.now()` in `cli.py` (must be absent).

## Plan samples are untrusted

The plan shows the command line but not the summary format or the exit codes. The `units=… buildings=… pages=… partial=…` line is this task's decision; the code mapping follows the plan's exit table (1 = data failure). Record both under `## Decisions`. Run `connex-watch --help` before documenting flags — `plan/stage-02-crawler.md` may name a flag that the repo spells differently.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
