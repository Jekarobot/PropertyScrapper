---
id: 08-01
stage: 08
title: Nightly script skeleton and dry-run
points: 1
owner: any
status: todo
depends_on: [04-15, 02-13]
files:
  - scripts/nightly.sh
  - pipeline/tests/test_nightly_script.py
---

# 08-01 · Nightly script skeleton and dry-run

**Points:** 1 · **Stage:** 08 — Hardening and release · **Owner lane:** any (ops) · **Depends on:** 04-15, 02-13 · **Unblocks:** 08-02

## Goal

Add `scripts/nightly.sh`: the absolute-path, single-instance, logged entry point whose `--dry-run` prints the four nightly commands and touches nothing.

## Why this task exists

Every stage so far produced a command an operator types by hand; nothing runs the pipeline unattended, and a half-written cron entry would crawl from whatever directory cron happens to use. One script must own path resolution, the
one-run-at-a-time rule and the log, because those are exactly what a scheduler cannot provide. Its dry-run is the contract the stub harness (08-04) and the runbook (08-11) quote, and the printed plan is the only description of the nightly a
reviewer can read without running it.

## Current state (preconditions)

- 02-13/02-14 are done: `connex-watch crawl --preset daily --resume --out DIR --run-id ID` works from any directory, exits 0/2/3/4, and `--resume` reopens an existing run directory.
- 03-13/03-14 and 04-14/04-15 are done: `build` takes `--raw`, `--out` and `--cache` and writes atomically; `check --dataset FILE` exits 0/1/2.
- `scripts/` and `pipeline/tests/test_nightly_script.py` do not exist; `pytest pipeline -q` is green and `logs/` is git-ignored.
- Confirm in `cli.py` that `--resume` with an explicit `--run-id` reopens that directory; if the shipped CLI ignores one of them, stop and report.

## Read before you start

- `CLAUDE.md` §3 (TDD), §5 (YAGNI), §6, §10 (never install a scheduler), §11 (commands).
- `plan/stage-08-hardening-ops.md` Task 8.1 items 1, 2, 7 and Traps — the `flock` caveat and "a second run is not an error".
- `plan/02-architecture.md` §6 P9, §7 (warm cache: under 4 min, under 80 requests) and §8 (politeness floor).
- `plan/90-test-matrix.md` rows O-01…O-04; `pipeline/src/connex_watch/cli.py` — the real flag names.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `scripts/nightly.sh` | strict-mode bash, path resolution, lock, log bootstrap, `--dry-run` |
| create | `pipeline/tests/test_nightly_script.py` | the six cases below plus one staging helper |

Nothing else: no `Makefile`, no `cli.py`, no README, no `.service`/`.timer`, no dependency.

## Test first (red)

Every case stages a throwaway repo under `tmp_path`: the real script copied into `<tmp>/repo/scripts/`, empty `data/` and `web/public/data/`, and a stub at `<tmp>/repo/.venv/bin/connex-watch` that appends its arguments to `<tmp>/stub.log`;
`CONNEX_WATCH_BIN` points at it. No case may touch the real repository, `data/` or venv.

1. `test_nightly_script.py::test_dry_run_prints_the_four_planned_commands_in_order` — `--dry-run` exits 0 with exactly four stdout lines, in this order: crawl with `--preset daily --resume --out data/raw --run-id nightly-<DATE>`; build with
   `--raw data/raw/nightly-<DATE> --out data/runs/<DATE>/dataset.json --cache data/cache/buildings.json`; check with `--dataset data/runs/<DATE>/dataset.json`; then the single publish line copying that dataset to `web/public/data/dataset.json`
   through `dataset.json.tmp`. Each line starts with the absolute stub path; `<DATE>` is one `YYYY-MM-DD` token — read it from line 1, require it in lines 2–4.
2. `::test_dry_run_creates_nothing` — every path under the staged repo is identical before and after: no `logs/`, no `data/runs`, no `data/cache`, no `web/public/data/dataset.json*`, no stub log.
3. `::test_dry_run_is_independent_of_the_working_directory` — the same four lines when invoked from `<tmp>` by absolute path and from the staged repo root; a relative `--out` must not resolve against the caller's directory.
4. `::test_second_run_while_the_lock_is_held_exits_zero_and_says_already_running` — a child holds `/tmp/connex-watch.lock` with `flock -n`; the run exits 0, stdout is empty, stderr contains `already running`; the holder is released in teardown.
   Pins O-03.
5. `::test_real_run_without_an_executable_binary_exits_two` — no `CONNEX_WATCH_BIN` and no `.venv/bin/connex-watch` → exit 2, one stderr line naming the path it looked for, `logs/` not created.
6. `::test_real_run_bootstraps_the_log_with_one_header_line` — stub present, no `--dry-run` → exit 0, `logs/nightly.log` first line contains the same `YYYY-MM-DD` token and the word `nightly`.

Expected red: `bash: scripts/nightly.sh: No such file or directory`, exit `127`, plus a `FileNotFoundError` from the subprocess call. A syntax error or a typo is not red.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_nightly_script.py -q`

## Implement (green)

- bash with `set -euo pipefail`; first statement `cd "$(dirname "$0")/.."`, then resolve `REPO_ROOT` physically. `LOG` and `BIN` are absolute and built from it; every CLI argument stays the documented relative path, so printed and executed lines
  are the same string.
- One date, `date +%F`, computed once: raw run id `nightly-<DATE>`, dataset directory `data/runs/<DATE>`. Two calls can straddle midnight and split the run id from the dataset directory.
- `BIN=${CONNEX_WATCH_BIN:-$REPO_ROOT/.venv/bin/connex-watch}`. Do not `source .venv/bin/activate`: cron and systemd (08-05) start with a minimal `PATH`.
- Lock before any work: open `/tmp/connex-watch.lock` on fd 9 and take `flock -n 9`; on contention print `already running` to stderr and exit 0.
- `--dry-run` prints the four lines and exits 0 before creating `logs/`, writing a log line or running anything; it must not require the binary to exist.
- Real run: require an executable `BIN` (else exit 2 naming it), `mkdir -p logs`, write one header line with the date token and `nightly`. Execution arrives in 08-02: the body ends with one placeholder line stated as such, exits 0, and must not
  be scheduled before 08-02.
- Unknown option → exit 2 with a one-line usage message on stderr. `--retention`, `--lock` and `--fail-check` are harness modes and belong to 08-04. One sentence at the top of the script states its contract; no other comments.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_nightly_script.py -q` — six green.

## Refactor / keep it DRY

The four lines come from one ordered description (crawl, build, check, publish) that `--dry-run` prints and 08-02 executes, so the plan cannot drift from the work. The stager and the stub live once in this test module; 08-02 and 08-03 extend
them.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_nightly_script.py -q
scripts/nightly.sh --dry-run
make test
```

Expected: six passed; the manual dry-run prints exactly four lines and `git status --short` stays clean; `make test` green. With `CONNEX_WATCH_BIN` unset the lines start with the real `.venv/bin/connex-watch`.

## Out of scope (do not do)

- Executing any step, exit-code propagation or the atomic publish (08-02); retention and rotation (08-03); the harness modes (08-04); systemd units (08-05); `make publish/nightly/help` (08-06).
- Installing, enabling or testing a cron job or timer (`CLAUDE.md` §10) — the owner's action, documented only.
- A pid file, a second lock, `--help` beyond the usage line, `--verbose`, retries, a logging library, or a real crawl in any test.

## Definition of done

- [ ] All six cases exist, named as written, and pass.
- [ ] Red run observed and quoted in the commit body (exit 127 or `FileNotFoundError`).
- [ ] Only `scripts/nightly.sh`, the test file and this task file changed (`git status --short`).
- [ ] `pytest pipeline -q` and `make test` green; no test touched the network or the real `data/`.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
08-01: add the nightly script skeleton and dry-run

- points: 1
- red: test_dry_run_prints_the_four_planned_commands_in_order failed (exit 127, no scripts/nightly.sh)
- add strict-mode nightly.sh with absolute paths, flock and the four-line dry-run
- add pytest cases that stage a throwaway repo and a stub binary
- suite: 91 passed
```

Files in the commit: `scripts/nightly.sh`, `pipeline/tests/test_nightly_script.py`, this task file. One commit, on trunk `main`.

## Traps and review notes

- Lock the descriptor, not a subshell: `flock -n 9` on an fd the script keeps open; in a subshell the lock is released before the crawl finishes and two runs overlap.
- `cd "$(dirname "$0")/.."` unquoted breaks on a path with spaces; resolve to absolute before the value reaches the log line.
- A dry-run that creates `logs/` or writes the header fails case 2; the only acceptable write is the lock file in `/tmp`.
- Do not scrape the run directory from the crawl's summary line — it does not carry the path, and `--run-id` exists so the build step can name it.
- Reviewers: `grep -c 'date +%F'` is 1; `grep -n 'source .*activate'` is empty; `already running` goes to stderr with exit 0.

## Plan samples are untrusted

`plan/stage-08-hardening-ops.md` Task 8.1 shows the four commands without `--run-id`, without a binary seam and without lock mechanics, and its build line predates the `--cache` flag from 04-14. Check each flag against `connex-watch build
--help` and `crawl --help`, and record the deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |

