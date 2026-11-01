---
id: 04-11
stage: 04
title: Enrichment report and progress logging
points: 0.5
owner: pipeline
status: todo
depends_on: [04-09]
files:
  - pipeline/src/connex_watch/enrich.py
  - pipeline/tests/test_enrich.py
---

# 04-11 · Enrichment report and progress logging

**Points:** 0.5 · **Stage:** 04 — Enrichment · **Owner lane:** pipeline · **Depends on:** 04-09 · **Unblocks:** nothing yet

## Goal

Freeze the `EnrichReport` field set and make the loop print its own progress line every 25 buildings, with counters that match the report.

## Why this task exists

A cold enrichment run takes twenty minutes. Without a line every few buildings the operator cannot tell a healthy run from a stalled one, and a silent loop is indistinguishable from a hang until it ends. The log line also has to agree with the report that lands in the dataset statistics: two independent counters that drift by one are worse than no log at all. Finally, logging belongs to this module — if the CLI prints progress, the same loop used by a test or another caller goes silent, and every future caller re-implements it.

## Current state (preconditions)

- 04-09 and 04-10 done: `enrich` with candidates, cache, cap, abort, `EnrichReport` carrying `fetched`/`hits`/`missing`/`capped`/`aborted`; thirteen cases green.
- `enrich.py` has no logger and no `merged_groups` field yet.
- Logging in this project is `logging` (module loggers, configured by the CLI); no `print` in library code.
- `pytest pipeline -q` is green.

## Read before you start

- `CLAUDE.md` §3 (assert the behaviour through `caplog`), §5 (no duplicated counters), §10 (never log page content).
- `plan/stage-04-enrichment.md` Task 4.3 — "Progress log every 25 buildings" and the exact key names.
- `plan/02-architecture.md` §4 — `logging` is the chosen logger; §3.3 — the dataset's `stats` block that consumes the report.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/enrich.py` | `merged_groups`, module logger, progress line |
| modify | `pipeline/tests/test_enrich.py` | the three new cases |

Nothing else may change: no CLI printing, no logging config, no merge logic (04-12).

## Test first (red)

1. `test_enrich.py::test_report_exposes_the_full_field_set` — `EnrichReport.model_fields` is exactly `fetched`, `hits`, `missing`, `merged_groups`, `capped`, `aborted`, and a fresh report has `merged_groups == 0`.
2. `...::test_progress_line_after_every_25_buildings` — 24 cold buildings produce **no** progress line; 50 produce exactly two, the first containing `ok=25`, `miss=0`, `cached=0`, `remaining=25`, the second ending with `remaining=0`. Assert on the numbers, not on a whole formatted sentence.
3. `...::test_progress_counters_match_the_report` — a mixed run (some hits, one content miss) → the last progress line's `ok`, `miss` and `cached` equal the report's resolved, missing and hits values.

Expected red: `AssertionError` on the field set (`merged_groups` missing) and an empty `caplog` record list for cases 2–3.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_enrich.py -q`

## Implement (green)

- `EnrichReport` gains `merged_groups: int = 0`. It stays 0 until 04-12 fills it; do not compute a placeholder.
- `enrich.py` gets a module logger via `logging.getLogger(__name__)`; no handler, no level, no `basicConfig`, no `print`.
- After every 25th processed building and only then, log one INFO line shaped `coords ok=<n> miss=<n> cached=<n> remaining=<n>`, where `ok + miss` is the number processed so far, `cached` is the hit count so far, and `remaining` is the number of input buildings not yet processed. The counters come from the same variables the report is built from — one source of truth.
- Log no URLs, no HTML, no coordinates per building: at INFO, a building name at DEBUG is acceptable, a page body never is.
- Do not log a final summary line here; the CLI prints the run summary from the report (04-14) so there is exactly one summary in a nightly log.
- Docstring: one sentence on the progress line's shape and interval.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_enrich.py -q` — sixteen green.

## Refactor / keep it DRY

The counters are incremented once and read by both the log line and the report builder; do not keep `ok`/`miss` as separate logger-only variables. The formatting lives in one small helper so a later change to the keys does not touch the loop body.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_enrich.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 16 passed, whole suite green, `ruff` exits 0. Manual check: run the small CLI enrichment with `-v` once 04-14 exists and confirm the line appears at INFO, not as a `print`.

## Out of scope (do not do)

- The CLI summary line, `--verbose`/`--quiet` flags, or configuring logging — 04-14 owns the CLI.
- A progress bar, `rich`, elapsed-time estimates, or per-building INFO lines.
- `merged_groups` accounting — 04-12.
- Writing the report to disk or into the cache.

## Definition of done

- [ ] All three cases exist, named as written, and pass; the thirteen earlier cases still pass.
- [ ] Red run observed and one failure line quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] Only the two files above plus this task file are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
04-11: log enrichment progress every 25 buildings

- points: 0.5
- red: test_report_exposes_the_full_field_set failed (AssertionError: merged_groups)
- add merged_groups to EnrichReport and a module logger
- emit coords ok/miss/cached/remaining from the report counters
- suite: N passed
```

Files in the commit: the two files above and this task file (status + Result). Nothing else.

## Traps and review notes

- Logging with `print` makes the line invisible to `caplog` and duplicates whatever the CLI prints; the module logger is the contract.
- A logger-only counter always drifts; assert equality with the report (case 3).
- Do not log the detail URL per building at INFO — a thousand lines turn a useful log into noise, and the URL contains the unit code.
- Never log a page body or the HTML length: it is third-party markup and the project rule is parse-and-discard.
- Reviewers: `grep -n "print(" pipeline/src/connex_watch/enrich.py` is empty; one `logger.info` call site.

## Plan samples are untrusted

The plan gives the key names and the interval in one sentence. Keep those names exactly; verify nothing else from it (there is no sample code for logging). Record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
