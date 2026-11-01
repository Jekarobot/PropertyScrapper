---
id: 08-03
stage: 08
title: Retention and log rotation
points: 0.5
owner: any
status: todo
depends_on: [08-02]
files:
  - scripts/nightly.sh
  - pipeline/tests/test_nightly_script.py
---

# 08-03 · Retention and log rotation

**Points:** 0.5 · **Stage:** 08 — Hardening and release · **Owner lane:** any (ops) · **Depends on:** 08-02 · **Unblocks:** 08-04 (retention mode), 08-11

## Goal

Prune `data/runs/*` to the newest 14 and `data/raw/*` to the newest 7 after a successful run, and rotate `logs/nightly.log` to `.1` when it has grown past 5 MB.

## Why this task exists

The nightly writes a ~1 MB dataset and a raw page tree every day; after three weeks the disk is full and the failure arrives as a half-written crawl at 02:30, far from the cause. Retention runs after a successful publish on purpose: pruning
before the run would delete the fallback dataset the "stale beats wrong" rule depends on. The log has the same shape — one growing file nobody reads until something breaks — so it rotates at a size that keeps a couple of weeks of history in one
file and one predecessor. Both operations delete files, which makes them the most dangerous lines in the repository: bounded to two globs under `data/`, listed before they act, tested only in throwaway trees.

## Current state (preconditions)

- 08-02 is done: the script runs crawl → build → check → publish, holds the lock on `/tmp/connex-watch.lock`, writes the header line into `logs/nightly.log`, and propagates exit codes.
- A real run produces `data/runs/<YYYY-MM-DD>/dataset.json` and `data/raw/nightly-<YYYY-MM-DD>/`; `logs/` is git-ignored.
- `pipeline/tests/test_nightly_script.py` has twelve green cases and the stager that copies the script into a throwaway repo.
- Nothing else writes to `data/runs` or `data/raw`, and both names are date-stamped, so name order is chronological.

## Read before you start

- `CLAUDE.md` §3 (the tests own this behaviour), §5, §6, §10 — deletions must never leave the two globs.
- `plan/stage-08-hardening-ops.md` Task 8.1 items 5–6 and Traps: "Retention deletes data — check the glob twice and dry-run once".
- `plan/02-architecture.md` §2 — the `data/` layout; `logs/` is generated and ignored.
- `plan/90-test-matrix.md` rows O-04 (keep 14 runs / 7 raw dirs) and O-06; `scripts/nightly.sh` — where the header is written.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `scripts/nightly.sh` | rotation before the header, retention after a successful publish, dry-run listing |
| modify | `pipeline/tests/test_nightly_script.py` | the five cases below and two tree helpers |

Nothing else: no `Makefile`, no `README.md`, no `data/` content in the repository, no new test file.

## Test first (red)

Every case builds its trees inside the staged throwaway repo, never in the real `data/`. Two helpers: one creating `n` run directories with ascending date names and a `dataset.json` in each, one creating `n` raw directories plus a stray
non-directory entry.

1. `test_nightly_script.py::test_retention_keeps_the_14_newest_run_directories` — 16 run directories created in reverse name order (oldest last) → after a successful run exactly the 14 newest names remain, the 2 oldest are gone, `data/runs`
   itself exists, and this run's published directory is present. Creating them out of order pins the ordering rule to the name, not to `mtime`.
2. `::test_retention_keeps_the_7_newest_raw_directories` — 9 raw directories plus a stray `data/raw/notes.txt` → the 7 newest directories remain, the 2 oldest are gone, `notes.txt` is untouched, and `data/raw/nightly-<DATE>` of the current run
   is kept.
3. `::test_rotation_renames_a_log_over_5_mb_before_writing` — pre-create a 6 MiB `logs/nightly.log` whose first line is `OLD-MARKER`, plus an older `.1` → after the run `logs/nightly.log.1` starts with `OLD-MARKER`, the new `logs/nightly.log`
   holds only this run's lines, and no third log file exists.
4. `::test_rotation_leaves_a_small_log_alone` — a 1 MiB log with a marker → the marker is still in `logs/nightly.log` afterwards and no `.1` was created; the sizes are far from the boundary, so the byte convention is not pinned.
5. `::test_dry_run_lists_the_deletions_on_stderr_and_removes_nothing` — 16 runs and 9 raw dirs, then `--dry-run` → stdout still exactly the four planned lines, stderr carries one `would delete <path>` line per doomed path (4 lines), and the tree
   is byte-identical afterwards. A real run logs one `delete <path>` line per removal before removing it.

Expected red: count assertions like `assert 15 == 14` (nothing pruned) and a missing `logs/nightly.log.1`. A failure caused by a missing fixture directory is broken, not red.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_nightly_script.py -q`

## Implement (green)

- Rotation first, immediately before the header: when `logs/nightly.log` exists and is larger than 5 MB, move it to `logs/nightly.log.1` (replacing a previous `.1`), then let the header create a fresh log. One generation only — no `.2`, no dates
  in the name, no compression.
- Retention after a successful publish, never before the crawl and never on a failed run: the failed run's directory is the evidence an operator needs, and a failed run must not destroy the fallback dataset's raw tree.
- Ordering: expand each glob, sort by name (`LC_ALL=C`), keep the first 14 (runs) and 7 (raw), delete the rest. Do not sort by `mtime`, which a restore or a copy rewrites.
- Bounds: only directory entries of `data/runs/*` and `data/raw/*`. Never a glob that can expand to the parent, never `rm -rf` on an unquoted variable, never a deletion inside `data/cache`, `web/` or `logs/`; guard that the resolved path starts
  with `$REPO_ROOT/data/` before removing.
- Empty globs are not an error (a fresh clone has neither tree): use `nullglob` so the script never tries to delete a literal pattern.
- Before each removal write one line — `delete <path>` into the log during a real run, `would delete <path>` to stderr during `--dry-run` — so the log explains a missing directory months later.
- `--dry-run` creates, moves and deletes nothing and keeps its four-line stdout contract from 08-01; the retention listing goes to stderr.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_nightly_script.py -q` — seventeen green.

## Refactor / keep it DRY

One prune function taking (glob, keep-count), called twice; one rotation block; one `date +%F`. The two tree helpers live once in the test module; 08-04's harness builds its own trees in shell — test data, not logic. Do not re-derive the current
run's directory in the pruner: it is kept by being the newest.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_nightly_script.py -q
scripts/nightly.sh --dry-run
make test
```

Expected: seventeen passed; the dry-run prints four stdout lines and no deletion lines on a repo with fewer than 14 runs; `make test` green. Manual check in a throwaway copy: create 16 run directories, run `--dry-run`, read the four `would
delete` lines, confirm `ls data/runs | wc -l` is still 16.

## Out of scope (do not do)

- A retention flag, a configurable keep-count, `--keep N`, compression, an archive directory, or date-based log rotation — YAGNI; the counts are the plan's.
- Pruning `data/cache`, `web/public/data`, other `logs/` files, or any path outside `data/runs` and `data/raw`.
- Deleting on a failed run, deleting the current run, or deleting when the glob is empty.
- Touching the real `data/` in a test, or committing anything under `data/` or `logs/`.

## Definition of done

- [ ] All five cases exist, named as written, and pass; the twelve earlier nightly cases still pass.
- [ ] Red run observed and quoted in the commit body.
- [ ] No test created or removed anything outside `tmp_path`; `git status --short` shows no `data/`.
- [ ] `pytest pipeline -q` and `make test` green.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
08-03: prune old runs and rotate the nightly log

- points: 0.5
- red: test_retention_keeps_the_14_newest_run_directories failed (assert 16 == 14)
- keep the newest 14 data/runs and 7 data/raw directories, listing each deletion first
- rotate logs/nightly.log to .1 above 5 MB, before the header is written
- suite: 102 passed
```

Files in the commit: `scripts/nightly.sh`, `pipeline/tests/test_nightly_script.py`, this task file. One commit, on trunk `main`.

## Traps and review notes

- The classic disaster is a glob that expands to nothing and an `rm -rf` on the literal pattern, or a variable that is empty when `rm -rf "$DIR"/` runs. Guard on `nullglob` and on a non-empty resolved path under `$REPO_ROOT/data/`.
- Do not sort by `mtime`: a `cp -a` or a restored backup rewrites it and the pruner would delete the newest tree. Names are date-stamped for exactly this reason.
- Rotating after the run instead of before it throws the current run's lines into `.1`; the header must land in the fresh file.
- Pruning before the publish wastes a night: if `check` then fails, the fallback dataset loses its sibling raw tree for no reason.
- Reviewers: `grep -n 'rm -rf'` shows one quoted, guarded removal per tree; `find … -delete` is absent; every deletion is preceded by its listing line.

## Plan samples are untrusted

The plan's one-liner `ls -d data/runs/* | head -n -14` is illustrative: `head -n -14` is a GNU extension and the pipeline lists an unquoted glob. Follow the documented counts (14 and 7) and this repository's shell, and record any deviation under
`## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |

