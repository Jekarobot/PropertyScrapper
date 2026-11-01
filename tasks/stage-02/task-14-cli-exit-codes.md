---
id: 02-14
stage: 02
title: CLI exit codes and operator messages
points: 1
owner: pipeline
status: todo
depends_on: [02-13]
files:
  - pipeline/src/connex_watch/cli.py
  - pipeline/tests/test_cli_crawl.py
---

# 02-14 · CLI exit codes and operator messages

**Points:** 1 · **Stage:** 02 — Crawler · **Owner lane:** pipeline · **Depends on:** 02-13 · **Unblocks:** 02-16, 08-02

## Goal
Make the crawl's outcome machine-readable: exit `0` with one summary line, `2` for a bad invocation, `3` when the site says no and `4` when the page structure broke — each with a message that tells the operator what to do next.

## Why this task exists
The nightly script (08-02) chains crawl → build → check and stops on the first non-zero status, and the runbook (08-11) explains each code. That works only if the codes are exact and failures are loud on stderr with the facts needed to act: which URL was refused, which page and field broke. Today the CLI either traces back or exits 0 regardless; this task is the last piece of the crawl's operator contract and changes no crawling behaviour.

## Current state (preconditions)
- 02-13 is done: `cli.main(argv)`, the `crawl` subcommand, preset resolution, `_make_fetcher`, the dry-run and a success path that prints a line and exits 0.
- 02-03 is done: `BlockedError` (with `.url`) and `FetchError` exist in `fetch.py`; stage 01 exposes `ParseError(component, position)` in `search_parser.py`, and the crawler raises it with `component="cards"` and the page number as `position` (02-07).
- `test_cli_crawl.py` has eight green cases and a local fake fetcher; 02-12 leaves a manifest unfinished when the crawl raises.
- The four codes `0`, `2`, `3`, `4` are the whole contract — no code 1, no code 5.

## Read before you start
- `CLAUDE.md` §3 (assert exact messages where they are the contract), §4, §6.
- `plan/01-discovery.md` §10 risks R1–R3 — which failure each code represents.
- `plan/02-architecture.md` §6 rules P2 and P4, §8 (stay read-only, stop when blocked).
- `plan/stage-02-crawler.md` Task 2.4 cases 3–5; `plan/90-test-matrix.md` row P-13.

## Files to touch
| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/cli.py` | exit-code mapping, summary line, messages |
| modify | `pipeline/tests/test_cli_crawl.py` | the five cases below |

Nothing else: no change to `crawl.py`, `store.py`, `fetch.py`, `README.md` or the Makefile.

## Test first (red)
Call `main([...])` in-process with `--out tmp_path`, `--run-id run-a`, the monkeypatched `cli._make_fetcher`, and `capsys` for both streams.

1. `test_cli_crawl.py::test_success_prints_one_summary_line_and_exits_zero` — the DEV fake, no flags → exit 0; stdout exactly one non-empty line; it starts with `pages=2 cards=32 avail=4 check=6 not=22 stop=empty_streak dup=0 elapsed=` and ends with `s`; the elapsed value is a non-negative float with one decimal; stderr empty.
2. `test_cli_crawl.py::test_bad_preset_exits_two_and_reports_on_stderr` — `--preset nope` → exit 2, stdout empty, stderr contains `nope`, `dev`, `daily`; usage errors go to stderr so a script capturing stdout gets a clean log.
3. `test_cli_crawl.py::test_blocked_site_exits_three_and_tells_the_operator_to_wait` — the fake raises `BlockedError` on the first request → exit 3; stderr contains `403`, the requested URL and the words `stop` and `later`; stdout empty; the run directory exists but its manifest still has `stopReason: null` and `finishedAt: null`.
4. `test_cli_crawl.py::test_parse_error_exits_four_naming_the_page_and_the_field` — page 1 = `search_dev_p1` (`1 Of 5`), page 2 = `search_dev_p2` with the card class renamed (`2 Of 5`) → exit 4; stderr contains `page 2` and `cards`; the manifest is unfinished; no `Traceback` text in either stream.
5. `test_cli_crawl.py::test_capped_run_reports_max_pages_and_exits_zero` — `--max-pages 1` on the DEV fake → exit 0; the summary contains `pages=1` and `stop=max_pages`; stderr empty; a cap is an operator choice, not a failure.

Expected red symptom: `assert 0 == 3` on the error cases (the CLI exits 0 or traces back today) and missing-line failures for the summary. A typo failure is not red.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_cli_crawl.py -q`.

## Implement (green)
- The handler catches exactly two types around the crawl: `BlockedError` → exit 3 and the stage-01 `ParseError` → exit 4. Everything else propagates: a `FetchError` that survived the retry policy, a bug or a `KeyboardInterrupt` must not be dressed up as one of the four outcomes.
- Exit 0 on a completed crawl (including `max_pages` and `empty_streak`), exit 2 for argparse errors and the unknown-preset path (02-13), 3 and 4 as above. `main` raises `SystemExit(code)`; it does not print and return.
- Messages, one line each on stderr, no traceback: blocked → status, URL and the instruction to stop and re-run later; parse → `page <exc.position>` and `field <exc.component>`. Use the error's attributes instead of re-parsing a message.
- The summary, on stdout, exactly `pages=<n> cards=<n> avail=<n> check=<n> not=<n> stop=<reason> dup=<n> elapsed=<s>s` — one space between fields, one line, printed once after success, never on failure. Counts come from `CrawlResult`; elapsed is a monotonic duration with one decimal.
- No colour, no emoji, no extra blank line, no logging on stdout: stdout is for the summary, stderr for problems.
- One-sentence docstring on the handler listing the four codes and their meaning.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_cli_crawl.py -q` — thirteen green.

## Refactor / keep it DRY
The four codes are constants or one small mapping referenced by the handler and the messages; no literal `3`/`4` scattered around. The summary formatter takes a `CrawlResult` and the elapsed value so 02-15 reuses the counts without copying the format. Re-run the suite.

## Verification (how to test it)
```text
. .venv/bin/activate && pytest pipeline/tests/test_cli_crawl.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 13 passed, the pipeline suite green, `ruff` exits 0. Manual check: `. .venv/bin/activate && connex-watch crawl --preset nope; echo $?` prints the preset list on stderr and `2`; `git status --short` shows no `data/` entry.

## Out of scope (do not do)
- A fifth exit code for `FetchError`, a timeout, a corrupt checkpoint or a full disk.
- Retrying inside the CLI, sleeping and retrying on `BlockedError`, falling back to a cached run.
- The exit-code table in `README.md` — 08-11 owns the runbook.
- The `-v` per-page lines and the `--delay` clamp — 02-15.
- JSON or machine-readable output, `--quiet`, log files, timestamps in the summary. Changing which exceptions the crawl raises (02-07 decided the `ParseError`).

## Definition of done
- [ ] All five cases exist, named as written, and pass; the eight 02-13 cases still pass.
- [ ] Red run observed and quoted in the commit body.
- [ ] Target tests + `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `cli.py`, `test_cli_crawl.py` and this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit
```text
02-14: map crawl outcomes to exit codes and messages

- points: 1
- red: test_blocked_site_exits_three_and_tells_the_operator_to_wait failed (assert 0 == 3)
- exit 3 on BlockedError and 4 on ParseError with actionable stderr lines
- print one summary line with pages, cards, states, stop reason, duplicates, elapsed
- suite: 81 passed
```

Files in the commit: `pipeline/src/connex_watch/cli.py`, `pipeline/tests/test_cli_crawl.py`, and this task file — nothing else.

## Traps and review notes
- Do not catch `Exception` and map "anything unexpected" to 4: a crawler bug must surface as a traceback and a failing nightly run, not a polite parse error.
- Do not finish the manifest after a failure; a blocked or broken run stays unfinished so stage 03 refuses it.
- Do not print the summary before the crawl completes, and never on stderr. No wall-clock timestamp in the summary — elapsed is a duration.
- Reviewer greps `print(` in `cli.py`: only the summary and the dry-run plan may write to stdout.

## Plan samples are untrusted
`plan/stage-02-crawler.md` gives the message intent, not exact strings; this task pins the facts each message must contain. Check the actual `BlockedError`/`ParseError` attributes in the repo before coding and record a deviation under `## Decisions` if a needed attribute does not exist.

## Decisions
<!-- append during implementation; leave empty at planning time -->

## Result
| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
