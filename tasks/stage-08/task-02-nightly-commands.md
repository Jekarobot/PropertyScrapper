---
id: 08-02
stage: 08
title: Crawl, build, check, publish sequence
points: 1
owner: any
status: todo
depends_on: [08-01]
files:
  - scripts/nightly.sh
  - scripts/tests/stub-connex-watch.sh
  - pipeline/tests/test_nightly_script.py
---

# 08-02 · Crawl, build, check, publish sequence

**Points:** 1 · **Stage:** 08 — Hardening and release · **Owner lane:** any (ops) · **Depends on:** 08-01 · **Unblocks:** 08-03, 08-04, 08-05, 08-06

## Goal

Make the nightly script run crawl → build → check → atomic publish, stop at the first failure, and pass the CLI's exit status through unchanged.

## Why this task exists

This is the product's unattended loop: without it the dataset never refreshes, and with a careless version a failed validation still overwrites a good dataset. The ordering is a safety property — a crawl that was blocked (exit 3) or a page whose
markup broke (exit 4) must stop before the build, and a `check` that rejects the dataset (exit 1) must stop before the publish, so the previous dataset stays live: stale beats wrong. The publish must be a rename, because the web app serves that
path while the copy happens. Every later task in this stage wraps this sequence, so its exit contract is the stage's spine.

## Current state (preconditions)

- 08-01 is done: the script resolves `REPO_ROOT`, takes the `flock` on `/tmp/connex-watch.lock`, bootstraps `logs/nightly.log`, accepts `--dry-run` and prints the four lines from one ordered description.
- 02-14/03-13/03-14/04-14/04-15 are done: crawl exits 0/2/3/4, build writes atomically and exits 2 on a corrupt cache, `check --dataset` exits 0/1/2.
- `pipeline/tests/test_nightly_script.py` has the six 08-01 cases, the `stage_repo` helper and a stub at `<tmp>/repo/.venv/bin/connex-watch`.
- The 08-01 placeholder line and the real steps are the only difference between the two commits; the dry-run output must stay byte-identical.

## Read before you start

- `CLAUDE.md` §3, §5 (no retry loop, no queue), §6, §10 (no live crawl in tests).
- `plan/stage-08-hardening-ops.md` Task 8.1 items 3–4 and 7, plus Traps: "a failed `check` must **not** publish; a stale dataset beats a wrong one".
- `plan/02-architecture.md` §6 P9 (the `check` failure conditions) and §3.3 (the published path and `stats.partial`).
- `plan/stage-02-crawler.md` exit table (0/2/3/4); `plan/stage-03-dataset-builder.md` Task 3.4 item 5 (0/1/2 for `check`); `cli.py` messages.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `scripts/nightly.sh` | the four executed steps, failure routing, atomic publish |
| create | `scripts/tests/stub-connex-watch.sh` | recorder stub shared by these tests and 08-04's harness |
| modify | `pipeline/tests/test_nightly_script.py` | the six new cases and the stub-driven helpers |

Nothing else: no `Makefile`, no `cli.py`, no retention (08-03), no README.

## Test first (red)

Extend the 08-01 stager: the stub records one line per invocation (`<subcommand> <arguments…>`) in `$CONNEX_WATCH_STUB_LOG`, creates the `--out`/`--run-id` directories and files the way the real CLI does, and exits with the code in
`CONNEX_WATCH_STUB_EXIT_<SUBCOMMAND>` when set, else 0. All cases run in a staged repo.

1. `test_nightly_script.py::test_four_steps_run_in_order_and_publish_the_new_dataset` — default run exits 0; the stub log holds exactly `crawl`, `build`, `check` in that order with the documented flags; `web/public/data/dataset.json` equals the
   build stub's bytes; stdout and stderr are empty.
2. `::test_failed_check_blocks_publish_and_keeps_the_previous_dataset` — a sentinel `web/public/data/dataset.json` exists first, `CONNEX_WATCH_STUB_EXIT_CHECK=1` → exit 1, the sentinel bytes are unchanged, no `dataset.json.tmp` exists, and the
   log shows `check` ran. This is O-02.
3. `::test_blocked_crawl_exits_three_and_runs_nothing_after_it` — stub crawl exits 3 → nightly exits 3, the log holds `crawl` only, no `data/runs` directory, nothing published.
4. `::test_parse_error_in_crawl_exits_four_unchanged` — stub crawl exits 4 → nightly exits 4, same "nothing after it" assertions; codes are propagated, never renumbered to a generic 1.
5. `::test_missing_run_directory_after_crawl_exits_one_without_building` — stub crawl exits 0 but creates no `data/raw/nightly-<DATE>` → exit 1, one stderr line naming the directory, `build` never invoked.
6. `::test_publish_replaces_the_target_without_leaving_a_tmp_file` — a pre-existing target with old bytes, successful run → new bytes, regular file, no `dataset.json.tmp`; a second consecutive run is idempotent.

Expected red: `assert 0 == 1` on case 2 and an empty stub log on case 1 (the script still ends in the 08-01 placeholder). Quote one of those lines.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_nightly_script.py -q`

## Implement (green)

- Execute the same ordered description `--dry-run` prints, one step at a time, with `>> "$LOG" 2>&1` on each command so the log holds the operator evidence; after each step inspect the status and stop immediately with `exit $status`. Never `set
  +e`, never `|| true`.
- Step 1 sends the crawl to the log and requires `data/raw/nightly-<DATE>` afterwards (exit 1 with a message if it is missing: a zero exit with no run directory means building from nothing).
- Step 2 builds `--raw data/raw/nightly-<DATE> --out data/runs/<DATE>/dataset.json --cache data/cache/buildings.json` and requires the file afterwards.
- Step 3 runs `check --dataset data/runs/<DATE>/dataset.json`; exit 1 stops before the publish and keeps the failed dataset and run directory for inspection (retention is 08-03).
- Step 4 publishes atomically: `mkdir -p web/public/data`, copy to `dataset.json.tmp` in that directory, rename over `dataset.json`. A publish failure exits 1 with one stderr line and leaves the previous dataset untouched because the rename
  never happened.
- Exit codes: 0 success (silent), 1 own failures and `check` failures, 2 passed through from a bad invocation or corrupt cache, 3 and 4 passed through from the crawl. No message rewriting — the CLI already logged the reason.
- Keep `--dry-run`, the lock and the header unchanged. One sentence per non-obvious step; no per-line comments.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_nightly_script.py -q` — twelve green.

## Refactor / keep it DRY

One step runner, one place mapping a status to an exit, one publish sequence, and the step description shared with `--dry-run` so no second copy of the flags exists. The stub is one file used by both the pytest cases and 08-04's harness — extend
it with a mode instead of forking a recorder.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_nightly_script.py -q
scripts/nightly.sh --dry-run
make test
```

Expected: twelve passed; the dry-run still prints the same four lines and creates nothing; `make test` green. Manual check in a throwaway copy: run the script with the stub and confirm stdout is empty while `web/public/data/dataset.json` holds
the stub's bytes.

## Out of scope (do not do)

- Retention, rotation and their dry-run listing (08-03); the harness modes and `scripts/test-nightly.sh` (08-04); `make` targets (08-06); scheduling (08-05).
- A retry loop, backoff, queue, a lock around the publish, rollback of a published dataset, run history, notifications, or a `--force` flag.
- Deleting or moving the failed dataset out of `data/runs/<DATE>` (retention keeps it; 08-03 prunes by count, never by success).
- Any change to `pipeline/**`, the CLI flags or the dataset schema.

## Definition of done

- [ ] All six cases exist, named as written, and pass; the six 08-01 cases still pass unchanged.
- [ ] Red run observed and quoted in the commit body.
- [ ] `pytest pipeline -q` and `make test` green; no case reached the network or the real `data/`.
- [ ] Only the three listed files and this task file changed (`git status --short`).
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
08-02: run crawl, build, check and publish in the nightly

- points: 1
- red: test_failed_check_blocks_publish_and_keeps_the_previous_dataset failed (assert 0 == 1)
- execute the four steps from the shared description and stop on the first failure
- publish through dataset.json.tmp and a rename; keep the previous dataset on any failure
- add the recording stub used by the tests and by 08-04
- suite: 97 passed
```

Files in the commit: `scripts/nightly.sh`, `scripts/tests/stub-connex-watch.sh`, `pipeline/tests/test_nightly_script.py`, this task file. One commit, on trunk `main`.

## Traps and review notes

- `set -e` plus a command in a pipeline hides the status; append to the log with a redirect on the command itself, not through `tee`.
- Copying straight onto `dataset.json` hands a half file to the browser and breaks "stale beats wrong" on a full disk — always `.tmp` then `mv`.
- `check` must run on the freshly built file, not on `web/public/data/dataset.json`; checking the published copy validates last night's data and always passes.
- Do not rewrite exit codes into 1 "for consistency": O-01/O-02 and the README exit table need 3 and 4 to reach the operator unchanged.
- Reviewers: `grep -n 'dataset.json.tmp'` shows exactly the copy and the rename; `grep -n '|| true'` is empty; case 2 asserts the old bytes, not only the exit code.

## Plan samples are untrusted

Task 8.1's block writes `build --raw <run dir>` without saying where the directory comes from, and it predates the `--cache` flag. Verify the four command lines against `connex-watch --help`; if `--resume` plus `--run-id` does not reopen the
same directory, stop and report rather than adding discovery heuristics.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |

