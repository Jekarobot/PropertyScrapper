---
id: 03-14
stage: 03
title: check subcommand and exit policy
points: 0.5
owner: pipeline
status: todo
depends_on: [03-12]
files:
  - pipeline/src/connex_watch/validate.py
  - pipeline/src/connex_watch/cli.py
  - pipeline/tests/test_validate.py
  - pipeline/tests/test_cli_build.py
---

# 03-14 · check subcommand and exit policy

**Points:** 0.5 · **Stage:** 03 — Dataset builder · **Owner lane:** pipeline · **Depends on:** 03-12 · **Unblocks:** 08-02, 08-06

## Goal

Add `check_dataset(ds, *, now=None)` for the P9 policy checks and `connex-watch check --dataset FILE`, which exits 0 for a publishable dataset and 1 with one line per violation otherwise.

## Why this task exists

The nightly script gates publishing on this command (`plan/90-test-matrix.md` O-02, O-05): a dataset that fails `check` must never reach the web app. Structural validation (03-12) is not enough — P9 also rejects an empty dataset (the herd guard), any unit without a url, a `partial` run, and a `generatedAt` older than 48 hours (a crawl that silently stopped). Printing one line per problem and exiting non-zero gives the operator a machine-checkable gate and a readable log entry at once.

## Current state (preconditions)

- 03-12 is done: `validate.py` has `validate_dataset(ds) -> list[str]` with nine passing tests.
- 03-13 is done: `cli.py` has the `build` subcommand, its exit codes and its summary formatting.
- `check_dataset` does not exist; `check` is not a subcommand.

## Read before you start

- `CLAUDE.md` §3 (TDD), §4 (0.5 point: 3–5 focused cases), §6.
- `plan/02-architecture.md` §6 P9 — the exact list of failing conditions, including the 48 h freshness rule.
- `plan/stage-03-dataset-builder.md` Task 3.4 item 5; `plan/90-test-matrix.md` rows O-05 and P-18.
- `plan/stage-08-hardening-ops.md` — `make publish` is gated on this exit code.
- `pipeline/src/connex_watch/cli.py` — the `build` handler's exception mapping and summary helper.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/validate.py` | `check_dataset` (policy layer over `validate_dataset`) |
| modify | `pipeline/src/connex_watch/cli.py` | the `check` subparser and its handler |
| modify | `pipeline/tests/test_validate.py` | `test_empty_dataset` (O-05) plus the staleness case |
| modify | `pipeline/tests/test_cli_build.py` | the four CLI cases below |

Nothing else may change: no build behaviour, no new dependency, no README edit (08-11 documents exit codes).

## Test first (red)

1. `test_validate.py::test_empty_dataset` — a dataset with `0` buildings and `0` units yields exactly one `check_dataset` violation containing `0 units` (the O-05 herd guard), and `stats.units` mismatches are not double-reported.
2. `test_validate.py::test_stale_generated_at_is_a_policy_violation` — with `now = 2026-11-03T21:45:00+00:00`, a `generatedAt` of `2026-11-01T21:45:00Z` (48 h exactly) passes and `…21:44:59Z` fails with a line containing `generatedAt` and the age; `partial` is irrelevant here.
3. `test_cli_build.py::test_check_cli_valid_dataset_exits_0` — `main(["check", "--dataset", tmp/"ds.json"])` on a run-mini dataset exits 0 and prints exactly `ok units=5 buildings=3`.
4. `test_cli_build.py::test_check_cli_prints_one_line_per_violation_and_exits_1` — a dataset mutated to have a duplicate unit id and a `0` price exits 1, prints exactly two lines on stdout, each naming its problem, with no traceback.
5. `test_cli_build.py::test_check_cli_partial_dataset_exits_1` — `stats.partial` true (otherwise valid) exits 1 with a line containing `partial` (P9).
6. `test_cli_build.py::test_check_cli_missing_dataset_exits_2` — a nonexistent `--dataset` path exits 2, names the path, and prints no violation lines.

Expected red: `SystemExit` code 2 for `check` (`invalid choice`) and `ImportError: cannot import name 'check_dataset'`.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_validate.py -q`

## Implement (green)

`check_dataset(ds: Dataset, *, now: datetime | None = None) -> list[str]` returns `validate_dataset(ds)` plus, in this order: `dataset has 0 units` / `dataset has 0 buildings` (one line each, never both when the dataset is empty of buildings only), one line per unit whose `url` is empty, `dataset is partial (<stopReason or page count>)`, and `generatedAt older than 48 h (<age>)`. `now` defaults to an aware UTC clock; the boundary is inclusive — exactly 48 h passes.

CLI: `check --dataset FILE` loads the file with the pydantic model, calls `check_dataset`, prints `ok units=N buildings=M` and exits 0 when the list is empty, otherwise prints every violation on its own line and exits 1. Unreadable file / invalid JSON → exit 2 naming the path. Reuse 03-13's summary formatter and its one place for exception mapping.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_validate.py -q` and
`. .venv/bin/activate && pytest pipeline/tests/test_cli_build.py -q`.

## Refactor / keep it DRY

`check_dataset` must call `validate_dataset`, not re-implement a structural check, and the CLI must not decide policy itself. The `ok units=… buildings=…` string comes from the same formatter as `build`'s summary. One clock parameter, one default, shared by the freshness check and its tests.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_validate.py -q
. .venv/bin/activate && pytest pipeline/tests/test_cli_build.py -q
pytest pipeline -q
connex-watch build --raw pipeline/tests/fixtures/run-mini --out /tmp/ds.json
connex-watch check --dataset /tmp/ds.json
ruff check pipeline
```

Manual check: the last two commands print the build summary and `ok units=5 buildings=3`; after `check`, `echo $?` is 0, and setting one price to `0` makes it print one line and exit 1.

## Out of scope (do not do)

- No fix-up, no rewrite of the dataset, no "repair and continue" mode.
- No `--strict`, no `--max-age` flag, no configurable thresholds (P9 is fixed; YAGNI), no HTTP/HEAD check of unit urls, no network, no new violation types beyond P9 and 03-12's list.
- No README exit-code table — 08-11 owns documentation.

## Definition of done

- [ ] All six cases exist, named as written, and pass; red run quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `validate.py`, `cli.py` and the two test files.
- [ ] `check_dataset` and the `check` handler each have a one-sentence contract.
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
03-14: add check subcommand and P9 policy

- points: 0.5
- red: test_empty_dataset failed (ImportError: cannot import name 'check_dataset')
- add check_dataset: empty dataset, url-less unit, partial, 48 h freshness
- add connex-watch check --dataset: ok line or one line per violation, exit 0/1
- suite: N passed
```

Files in the commit: `validate.py`, `cli.py`, `test_validate.py`, `test_cli_build.py` and this task file. Nothing else.

## Traps and review notes

- Printing violations to stderr breaks the nightly log grep; stdout only, one line each.
- The 48 h boundary is inclusive and must be tested on both sides, otherwise a slow nightly run fails randomly.
- Empty dataset: report 0 units **or** 0 buildings, not both twice — the O-05 row counts lines; `check` never rewrites the file it validates.
- Reviewers: grep `cli.py` for a second `validate_dataset(` call — the policy belongs in `validate.py`.

## Plan samples are untrusted

The plan lists P9 conditions in prose only. The `ok units=… buildings=…` line and the exit codes 0/1/2 are this task's contract; record them under `## Decisions`. Do not add a dependency for freshness arithmetic — stdlib `datetime` is enough.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
