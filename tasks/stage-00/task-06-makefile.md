---
id: 00-06
stage: 00
title: Makefile with test/check entry points
points: 1
owner: pipeline
status: todo
depends_on: [00-03, 00-05]
files:
  - Makefile
---

# 00-06 · Makefile with test/check entry points

**Points:** 1 · **Stage:** 00 — Bootstrap · **Owner lane:** pipeline · **Depends on:** 00-03, 00-05 · **Unblocks:** 05-01, 08-06

## Goal

Add one repo-root `Makefile` with the seven targets later stages use, skipping the web half until `web/` exists.

## Why this task exists

Later stages, the owner and the reviewer all need one command that proves the toolchain; without it each lane invents
its own invocation and stage 08 retrofits `publish`/`nightly`. `web/` does not exist until 05-01, so the web steps are
guarded rather than failing the check, and these targets are the entry point the README (00-02) and 08-11 point at.

## Current state (preconditions)

- 00-03 and 00-05 are done: `.venv` has pytest and ruff, `pytest pipeline -q` is 20 passed and `ruff` is clean. `Makefile`,
  `web/` and `scripts/` do not exist; GNU Make 4.3 and pnpm 12.16.1 are installed (re-check, do not assume).

## Read before you start

- `plan/stage-00-bootstrap.md` Task 0.4 — the target table and the "guard with `[ -d web ]`" instruction.
- `plan/stage-08-hardening-ops.md` Task 8.3 — final `publish`/`nightly`/`help` semantics and the offline `check`;
  `CLAUDE.md` §11 (exact commands) and `plan/90-test-matrix.md` O-06 (clean clone, no dataset must pass).

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `Makefile` | the seven targets, `.PHONY`, `.DEFAULT_GOAL` |

## Test first (red)

No pytest file here: this is build glue, the scaffolding case in `CLAUDE.md` §3. The red is observed as failing `make`
invocations with exact exit codes.

1. `C1 make check` — today exits `2` with `make: *** No rule to make target 'check'.  Stop.`; `make test` and `make help` fail the same way (`C2`, `C3`).
2. `C4 make test` after — exits `0`, runs pytest once and prints the one-line web skip message.
3. `C5 make check` after — exits `0`, runs ruff then pytest, and the web steps only when `web/package.json` exists.
4. `C6 make help` after — exits `0` and lists exactly `test`, `check`, `crawl`, `build`, `publish`, `nightly`, `help`.
5. `C7 make publish` after — exits non-zero and names the missing dataset; `C8 make -n crawl` prints the crawl command
   without running it.

Expected red: `make check` exits `2` with the no-rule line. Run `. .venv/bin/activate` first.

## Implement (green)

`.DEFAULT_GOAL := help`, every target `.PHONY`, recipes run from the repo root and call `pytest`, `ruff`, `pnpm` and
`connex-watch` from `PATH`; invoke with `. .venv/bin/activate` then `make <target>`, never via a hardcoded `.venv/bin`
path. `help` prints one line per target and needs neither the venv nor `web/`.

`test`: `pytest pipeline -m "not network" -q`, then the guarded `pnpm -C web test --run`. `check`: `ruff check pipeline`,
the same pytest invocation, then guarded `pnpm -C web typecheck` and `pnpm -C web test --run`. One shell test for
`web/package.json` guards every web step: absent → print `web/ not created yet — skipped (05-01)` and continue; present
→ a pnpm failure propagates.

`crawl`: `connex-watch crawl --preset daily` (CLI in 02-13); `build`: `connex-watch build` until 03-13/04-14 define flags;
`publish`: copy the newest `data/runs/*/dataset.json` to `web/public/data/dataset.json`, exiting `1` when no run or no
`web/` exists; `nightly`: `scripts/nightly.sh`, erroring clearly when missing (08-01). `make check` stops at the first
failure: no `-` prefixes, no `|| true`, no `;` chain hiding an exit code. Nothing touches the network, the fixtures stay
frozen and only `Makefile` is committed.

## Refactor / keep it DRY

One definition per command: the pytest line in `pipeline-test`, the web guard in one construct used by both web steps.

## Verification (how to test it)

```text
. .venv/bin/activate
make test
make check
make publish; echo "exit=$?"
```

Expected: `test`/`check` exit `0` with `20 passed` and the skip line; `publish` fails with the no-dataset message, and
`make` reports `exit=2` for that recipe's `1`.

## Out of scope (do not do)

- Final `publish` and the `connex-watch check` step in `check` — 08-06; `scripts/nightly.sh` — 08-01; systemd, cron,
  Docker, CI, `dev`/`install`/`fmt` targets; creating `web/` (05-01); committing a dataset.
- Editing `README.md`, `pipeline/**` or `web/**`; running `crawl`, `build` or `nightly` before their tasks land.

## Definition of done

- [ ] C1–C8 behave as written, including the exit codes and the skip message.
- [ ] `make test` and `make check` are green from the repo root with the venv active; red quoted in the commit body;
      only `Makefile` is added; `status: done`; `## Result` filled; exactly one commit on the trunk.

## Commit

```text
00-06: add make targets for test and check

- points: 1
- red: make check exited 2 (make: *** No rule to make target 'check'.  Stop.)
- add the seven targets and guard every web step on web/package.json until 05-01
- suite: 20 passed via make check
```

Files in the commit: `Makefile` and this task file. One task = one commit, alone on the trunk (`main`).

## Traps and review notes

- Recipe indentation must be a tab, and each recipe line is its own shell: a `cd`/`source` does not persist, so never
  put `source .venv/bin/activate` inside a recipe — document the activation instead.
- Guard on `web/package.json`, not on `web/`: an empty directory must skip and a present package must propagate failure.
- Never swallow a failure with `-`, `|| true` or a `;` chain; `make check` must exit non-zero on any failure.
- Keep `-m "not network"` in both pytest invocations; use `make -n` for targets whose tools do not exist yet, and
  `grep -n 'web/' Makefile` to confirm every web line is guarded.

## Plan samples are untrusted

Task 0.4 gives a target table with an ellipsis for `build`: it is a contract, not recipe text. Apply `CLAUDE.md` §6,
check `make --version`, verify with `make -n`, and never invent flags 03-13/04-14 have not defined.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
