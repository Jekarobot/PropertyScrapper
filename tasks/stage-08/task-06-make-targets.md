---
id: 08-06
stage: 08
title: publish, nightly, help and the final check
points: 1
owner: any
status: todo
depends_on: [08-02, 03-14, 04-14]
files:
  - Makefile
---

# 08-06 · publish, nightly, help and the final check

**Points:** 1 · **Stage:** 08 — Hardening and release · **Owner lane:** any (ops) · **Depends on:** 08-02, 03-14, 04-14 · **Unblocks:** 08-12

## Goal

Complete the command surface: `publish` from the newest run, `nightly` through the script, `help` with one line per target, and a `check` that also validates the published dataset — skipping that step, loudly, on a fresh clone.

## Why this task exists

`make check` is the only command the owner, a reviewer and a fresh agent all run, and today it stops before the dataset: it proves the code compiles and the tests pass, not that the file the browser would fetch is publishable. Closing that gap
is what makes O-06 and P9 meaningful outside the nightly — a stale or partial dataset fails `make check`, and a clone with no dataset still passes because there is nothing to validate. `publish` is the manual counterpart for the day the timer
did not run, `nightly` is the one-word entry point the README tells people to use, and `help` exists because the Makefile now has seven targets. This is the last task before the release pass.

## Current state (preconditions)

- 00-06 is done: `Makefile` has `.DEFAULT_GOAL := help`, `.PHONY` targets `test`, `check`, `crawl`, `build`, `publish`, `nightly`, `help`, and guards every web step on `web/package.json`.
- 08-02 is done: `scripts/nightly.sh` exists and is executable. 03-14 is done: `check --dataset FILE` exits 0/1/2 and prints `ok units=N buildings=M` or one line per violation. 04-14 is done: `build` writes `data/runs/<date>/dataset.json`.
- `web/public/data/dataset.json` and `data/runs/` are git-ignored and may be absent.
- `make test` and `make check` are green from the repo root with the venv active.

## Read before you start

- `CLAUDE.md` §3 (build glue is the scaffolding exception — the red is observed as failing `make` invocations), §5 (one definition per command), §6, §11 (the exact pytest/pnpm/ruff invocations).
- `plan/stage-08-hardening-ops.md` Task 8.3 — the four targets and the dataset-skip rule.
- `plan/02-architecture.md` §6 P9 — the exact failure list `check` enforces; `plan/90-test-matrix.md` rows O-06 and P-09.
- `Makefile` — how the web guard and the tool lines are written today; extend it, do not rewrite it.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `Makefile` | `publish` hardening, `nightly`, `help` descriptions, the dataset step in `check` |

Nothing else: no `scripts/`, no `README.md`, no `pipeline/**`, no `web/**`, no target beyond the seven, no dependency.

## Test first (red)

No test file: build glue (`CLAUDE.md` §3). The red is observed as failing invocations with exact exit codes; record them, then implement.

1. `check-1` `make check` on a tree without `web/public/data/dataset.json` — before, exits 0 but never mentions the dataset; after, exits 0 and prints `dataset check skipped: web/public/data/dataset.json not found (run make nightly or make
   publish)`. A fresh clone must pass: O-06.
2. `check-2` `make check` with a valid published dataset — after, runs `connex-watch check --dataset web/public/data/dataset.json` as the last step and exits 0, with the CLI's `ok units=… buildings=…` line visible.
3. `check-3` `make check PYTEST=false` — after, exits non-zero at the pytest step and never reaches the pnpm or dataset steps; this is why each tool is one overridable variable used once.
4. `check-4` `make publish` with an empty `data/runs/` — after, exits 1 and names `data/runs` as the missing directory, creating nothing.
5. `check-5` `make publish` with two runs — after, `web/public/data/dataset.json` equals the newest run's file byte for byte, `web/public/data/` is created when absent, no `dataset.json.tmp` remains, and a second publish is idempotent.
6. `check-6` `make nightly` — after, invokes `scripts/nightly.sh` from the repo root; `make -n nightly` prints that one command, and removing the script makes the target fail with a message naming it (keep 00-06's guard).
7. `check-7` `make help` — after, prints exactly seven lines, one per target, each `<target>` then a one-sentence description, in the order test, check, crawl, build, publish, nightly, help; no target lacks a description and none spans two
   lines.

Expected red: `make check` has no dataset line; `make publish` exits 1 with the old wording; `make help` shows no descriptions. Quote the exact missing line.

## Implement (green)

- One variable per tool (`PYTEST`, `RUFF`, `PNPM`, `CONNEX_WATCH`) defined once and used once per target, so the failure-injection checks work and no command string is duplicated between `test` and `check`.
- `check` order: `ruff check pipeline`, `pytest pipeline -m "not network" -q`, guarded `pnpm -C web typecheck`, guarded `pnpm -C web test --run`, then the dataset step. Every step is its own recipe line with no `-` prefix, no `|| true`, no `;`
  chain hiding a status; the dataset step is one shell test on the file with the skip message on the absent branch and the `connex-watch check --dataset` call on the present branch.
- `publish`: pick the newest `data/runs/*/dataset.json` in name order (directories are date-stamped), exit 1 naming `data/runs` when there is no run or no dataset, `mkdir -p web/public/data`, copy to `dataset.json.tmp`, rename over
  `dataset.json`. Same temp name and sequence as `scripts/nightly.sh`; do not invent a second scheme.
- `nightly`: call `scripts/nightly.sh` from the repo root, failing clearly when the script is missing (00-06's guard stays). Pass no flags; the script owns its arguments.
- `help`: one line per target, name first, then a one-sentence description; `.DEFAULT_GOAL` stays `help`. `test` keeps running pytest and the guarded web tests, and the nightly harness from 08-04 stays where it is. No target may run `crawl`,
  `build` or `publish` as a side effect.

Run: `make help`, `make check`, `make publish` with and without runs, `make check PYTEST=false` — all seven checks behave as written.

## Refactor / keep it DRY

One tool variable each, one web guard, one dataset-skip message, one publish sequence. The `.tmp` + rename rule now exists in exactly two files (this Makefile and `scripts/nightly.sh`); keep the temp name identical so `grep -rn
'dataset.json.tmp'` finds both and a reviewer sees they agree. Do not extract a helper script for two identical lines.

## Verification (how to test it)

```text
make help
make check
make publish ; echo "exit=$?"
make check PYTEST=false ; echo "exit=$?"
make test
```

Expected: seven help lines; `check` green with the skip message on a tree without a dataset; `publish` exits 1 and names `data/runs` when there is none, and copies the newest run when there is one; `make check PYTEST=false` exits non-zero before
the pnpm steps; `make test` green.

## Out of scope (do not do)

- New targets (`dev`, `install`, `fmt`, `e2e`, `nightly-check`), a `Makefile` include, or moving logic into a helper script.
- Publishing on a schedule, running `check` inside the publish target, or requiring the dataset for `make check` (a fresh clone must pass).
- Editing `scripts/nightly.sh` (08-02/08-03/08-04 own it), `README.md` (08-05/08-11) or any test.
- Changing the pytest invocation away from `-m "not network"`; the network test stays opt-in.

## Definition of done

- [ ] Checks 1–7 behave as written, with before/after evidence quoted in the commit body.
- [ ] `make check` passes with and without `web/public/data/dataset.json`, and fails non-zero when any step fails.
- [ ] `make help` lists exactly the seven targets, one line each.
- [ ] Only `Makefile` and this task file changed (`git status --short`); no `data/` or `web/public/` file is committed.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and the check outputs.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
08-06: finish publish, nightly, help and check

- points: 1
- red: make check printed no dataset step and make help printed no descriptions
- add the dataset step to check with a clear skip on a fresh clone
- publish the newest run atomically; run scripts/nightly.sh from make nightly
- suite: 17 nightly cases + 6 harness modes, make check green
```

Files in the commit: `Makefile` and this task file. One commit, on trunk `main`.

## Traps and review notes

- Guarding the dataset step with `test -f … || exit 0` passes on a fresh clone but also hides a corrupt dataset; the branch must be explicit, print the skip message, and let a non-zero `check` abort `make`.
- `$(shell ls data/runs/*)` evaluates at parse time and on every target, including `help`; resolve the newest run inside the `publish` recipe instead.
- A `publish` that copies onto `dataset.json` directly leaves a half file for the browser; the temp file must sit in the same directory so the rename stays atomic.
- `make check` must not depend on `data/runs/` existing; only `web/public/data/dataset.json` matters.
- Reviewers: no recipe line starts with `-` or hides a status; `grep -c 'dataset.json.tmp'` finds two agreeing files; `make -n check` shows five steps in the documented order.

## Plan samples are untrusted

Task 8.3 spells the final step as `pnpm -C web test` and names targets 00-06 already created. Verify the actual invocation with `make -n` and against `CLAUDE.md` §11, keep the repository's `--run` form, and record any deviation under `##
Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |

