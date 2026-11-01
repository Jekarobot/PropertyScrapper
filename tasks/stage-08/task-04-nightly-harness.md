---
id: 08-04
stage: 08
title: Stub-binary harness for the nightly
points: 2
owner: any
status: todo
depends_on: [08-02]
files:
  - scripts/test-nightly.sh
  - scripts/tests/stub-connex-watch.sh
  - Makefile
  - scripts/nightly.sh
---

# 08-04 · Stub-binary harness for the nightly

**Points:** 2 · **Stage:** 08 — Hardening and release · **Owner lane:** any (ops) · **Depends on:** 08-02 (and 08-03 for the retention mode) · **Unblocks:** 08-12

## Goal

Add `scripts/test-nightly.sh`, the offline harness that proves O-01…O-04 — four steps in order, a failing `check` blocking the publish, a second concurrent run doing no work, and retention — by pointing `CONNEX_WATCH_BIN` at a recorder stub in
throwaway directories.

## Why this task exists

Shell is the one part of the system no Python test exercises end to end, and the nightly is the part nobody watches. A harness with a stub binary is the only way to prove ordering and the publish gate without a live crawl: the stub records what
was invoked, so a "success" that skipped the check cannot pass. It must also be able to fail — a harness that always prints `ok` is worse than none — so one mode deliberately breaks the gate and requires the harness to notice. `make test` runs
it, which gates every later change to the script and gives the release pass (08-12) its command. The matrix names this file as the evidence for O-01…O-04.

## Current state (preconditions)

- 08-02 is done: the script executes crawl → build → check → publish, stops at the first failure, publishes through `dataset.json.tmp`, and resolves the binary from `CONNEX_WATCH_BIN`.
- 08-03 is done: retention keeps the newest 14 runs and 7 raw directories. The declared dependency is 08-02, but the `--retention` mode asserts 08-03's behaviour, so run 08-03 first if it is not merged; the files do not overlap.
- `scripts/tests/stub-connex-watch.sh` exists from 08-02 with the argv recorder and the per-subcommand exit override.
- `Makefile` has `test` and `check` from 00-06; `test` does not call this harness yet.
- `pytest pipeline/tests/test_nightly_script.py` is green (seventeen cases).

## Read before you start

- `CLAUDE.md` §3 (never weaken an assertion to go green), §4 (2 points: an integration with a stub), §6, §10 (no live crawl), §11.
- `plan/90-test-matrix.md` rows O-01…O-04 — the exact four rows and their documented commands.
- `plan/stage-08-hardening-ops.md` Task 8.1 "TDD-ish verification" and Traps; `plan/02-architecture.md` §7 — a warm nightly under 4 minutes, so the harness stays under a few seconds; read `scripts/nightly.sh` and the stub before extending.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `scripts/test-nightly.sh` | modes, staging, assertions, summary, exit status |
| modify | `scripts/tests/stub-connex-watch.sh` | only if a mode needs a knob the pytest cases do not |
| modify | `Makefile` | `test` runs the harness after pytest |
| modify | `scripts/nightly.sh` | only a defect a mode exposes — never to make a weak assertion pass |

Nothing else: no pytest changes, no README, no systemd, no new dependency.

## Test first (red)

The harness is the test artifact; its own red is observed in three steps.

- First red: `scripts/test-nightly.sh` exits `127` with `No such file or directory`, and `make test` passes without it — a suite that cannot see the nightly.
- Second red: write the modes first as bare failing checks (each prints `not ok` and exits 1), run `scripts/test-nightly.sh --fail-check`, quote the `not ok` line.
- Mutation proof: after the modes are green, copy the script, delete the `check` gate and run the harness against the copy — `--fail-check` must fail loudly. A harness that survives that is vacuous.

The six cases (one per mode):

1. `order` (default run) — the stub log shows exactly `crawl`, `build`, `check` in that order with the documented flags, and the published `dataset.json` holds the build stub's bytes; prints `ok order`. O-01.
2. `--fail-check` — `CONNEX_WATCH_STUB_EXIT_CHECK=1` with a sentinel dataset already published → nightly exits 1, the sentinel bytes are unchanged, no `dataset.json.tmp` remains; prints `ok fail-check`. O-02.
3. `--lock` — a child holds `/tmp/connex-watch.lock`; the second invocation exits 0, prints `already running`, appends nothing to the stub log and creates no run directory; prints `ok lock`. O-03, the only two-process case.
4. `--retention` — 16 run directories and 9 raw directories before the run → 14 and 7 after it, the oldest gone, the current run's raw directory present; prints `ok retention`. O-04.
5. `--mutation-gate` — copy the script, remove the `check` gate, run `--fail-check` against the copy: the harness must exit non-zero and print a `not ok` line naming the mode.
6. `--offline`, also asserted in every mode — `CONNEX_WATCH_BIN` points inside the temp directory and `.venv/bin/connex-watch` was never invoked, so no mode can reach the site.

Expected red: exit 127 on the first run, then `not ok fail-check` before the modes exist; a mutation run that stays green fails this task.

Run: `scripts/test-nightly.sh` — five `ok` lines (six with `--mutation-gate`), exit 0.

## Implement (green)

- bash, `set -euo pipefail`, `mktemp -d` with a cleanup trap; copy the script into `<tmp>/repo/scripts/`, create `data/runs`, `data/raw`, `web/public/data`, `.venv/bin/`, point `CONNEX_WATCH_BIN` at the stub. Nothing may touch the real repository, `data/` or venv.
- One function per mode, one shared `run_nightly` wrapper that resets the stub log and asserts the exit status; each mode prints one `ok <mode>` or one `not ok <mode>: <reason>` and exits 1 immediately.
- No arguments runs every mode except the mutation one; `--fail-check`, `--lock`, `--retention` and `--mutation-gate` select one; an unknown flag prints usage and exits 2.
- The lock case starts the holder with the `flock` utility in the background, waits until the lock is demonstrably held, and kills it in a trap on every exit path; a leaked holder makes every later mode report `already running`.
- No `sleep` as synchronisation: poll the stub log or a file with a bounded loop so the harness stays fast and cannot flake.
- The harness never asserts on timings or on a date it did not compute; it reads the date token from the files the scripts produced.
- `Makefile`: `test` runs the harness after pytest, before the guarded web steps, fail-fast like the other steps, and prints its summary line so its presence is visible.
- If a mode exposes a real defect in `scripts/nightly.sh`, fix the script in this commit; never relax the assertion.

Run: `scripts/test-nightly.sh` — five `ok` lines (six with `--mutation-gate`); `make test` green.

## Refactor / keep it DRY

The stub, the staging layout and the assertions that already exist as pytest cases must not be re-implemented: this harness drives the same stub and asserts the matrix-level behaviour (two processes, real trees, byte comparison). If a case is
fully expressible as a pytest case, delete it here rather than keeping two homes.

## Verification (how to test it)

```text
scripts/test-nightly.sh
scripts/test-nightly.sh --fail-check
scripts/test-nightly.sh --lock
scripts/test-nightly.sh --retention
make test
```

Expected: each invocation prints its `ok` line and exits 0; `--mutation-gate` exits non-zero by design against a broken copy; `make test` prints the harness summary and stays green. Runtime under five seconds.

## Out of scope (do not do)

- A CI workflow, a Docker image, `shellcheck` as a new dependency, or a shell test framework; testing the CLI itself (stages 02–04 own that), the dataset beyond the stub's bytes, or the web app.
- Installing the timer to "test it for real" (`CLAUDE.md` §10), or any live crawl, even opt-in.
- Retrying a flaky mode, or `set +e` around a mode to collect all failures.

## Definition of done

- [ ] All five modes exist, are named as written, and pass; the mutation gate is observed failing.
- [ ] Red run (exit 127, then `not ok`) observed and quoted in the commit body.
- [ ] `make test` runs the harness and fails when a mode fails; only the four listed files and this task file changed (`git status --short`); no real `data/`.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash, mode list and runtime.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
08-04: add the stub-binary harness for the nightly

- points: 2
- red: scripts/test-nightly.sh exited 127 (no such file); not ok fail-check before the modes existed
- add test-nightly.sh with order, fail-check, lock, retention, mutation and offline modes
- wire the harness into make test; keep one stub shared with the pytest cases
- suite: 17 nightly cases + 6 harness modes
```

Files in the commit: `scripts/test-nightly.sh`, `scripts/tests/stub-connex-watch.sh`, `Makefile`, `scripts/nightly.sh` (only if a defect was fixed), this task file. One commit, on trunk `main`.

## Traps and review notes

- The lock mode is the one that flakes: wait for the holder (poll after starting `flock`) and release it in a trap; a leaked holder makes every later mode say `already running`. A green harness proves nothing unless the mutation gate is red, so paste both outputs in `Result`.
- Do not run the harness against the repository root: copy the script into the temp tree so its `cd` cannot reach the real `data/`, and assert that path in `--offline`.
- `mktemp -d` under `/tmp` is fine; a cleanup trap that runs `rm -rf "$TMP"` with the variable unset is the one dangerous line in the file — guard it.
- Reviewers: no `sleep`; one `ok` per mode, each with an assertion behind it; `grep -n 'test-nightly' Makefile` is one line in `test`.

## Plan samples are untrusted

The plan only sketches this harness ("runs the script with `CONNEX_WATCH_BIN` pointing at a stub … asserts the four commands are invoked in order") and predates the retention mode's dependency on 08-03. Treat the matrix's four rows as the
contract, verify the flags against the shipped script, and record deviations under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |

