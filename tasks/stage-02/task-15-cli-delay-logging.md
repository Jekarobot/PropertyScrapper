---
id: 02-15
stage: 02
title: Delay clamp and per-page verbose logging
points: 0.5
owner: pipeline
status: todo
depends_on: [02-13]
files:
  - pipeline/src/connex_watch/cli.py
  - pipeline/src/connex_watch/crawl.py
  - pipeline/tests/test_cli_crawl.py
---

# 02-15 · Delay clamp and per-page verbose logging

**Points:** 0.5 · **Stage:** 02 — Crawler · **Owner lane:** pipeline · **Depends on:** 02-13 · **Unblocks:** 08-02

## Goal
Enforce the politeness floor on `--delay` with a visible warning, and make `-v` print exactly one line per parsed page with that page's state counts and the running request number.

## Why this task exists
`--delay` could quietly turn the crawler into a flood, so architecture §8 makes the 1.0 s floor non-negotiable: a smaller value is corrected **and** announced, never silently honoured or silently ignored. The per-page line is the diagnostic that catches the site changing shape — a page that used to be all-available and is suddenly all-`check_availability` is the earliest signal in `plan/01-discovery.md` §6, and it is only visible if every page reports its own counts. Both are what the runbook tells the owner to look at when a run stops early.

## Current state (preconditions)
- 02-13 is done: `cli.main`, the `crawl` subcommand, `--delay` (default `1.2`) and `_make_fetcher(min_interval)` exist; `test_cli_crawl.py` is green.
- 02-07 introduced the module logger in `crawl.py` for the lying-pager warning; no INFO records are emitted yet.
- 02-14 prints one summary line on stdout and nothing else; failures use codes 2/3/4 on stderr.
- The loop appends one `PageResult` per parsed page to `result.pages`, in fetch order.

## Read before you start
- `CLAUDE.md` §3 (no real time in tests), §5 (stdlib logging only), §6.
- `plan/02-architecture.md` §8 — the `--delay` floor of `1.0 s` is enforced in code.
- `plan/01-discovery.md` §6 — the per-page state histogram as the early-warning signal.
- `plan/stage-02-crawler.md` Task 2.4 cases 6 and 7; `plan/90-test-matrix.md` rows P-13 and P-11.

## Files to touch
| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/cli.py` | delay clamp, warning, `-v` logging setup |
| modify | `pipeline/src/connex_watch/crawl.py` | one INFO record per parsed page |
| modify | `pipeline/tests/test_cli_crawl.py` | the four cases below |

Nothing else: not `fetch.py` (its floor is 02-01's), not the summary format (02-14), not the manifest.

## Test first (red)
Reuse the DEV fake and `--out tmp_path`. Record the exact factory argument in the monkeypatched `_make_fetcher`.

1. `test_cli_crawl.py::test_delay_below_the_floor_is_clamped_with_a_warning` — `--delay 0.2` → the factory received `1.0`; stderr contains one warning line mentioning `0.2` and `1.0`; exit 0 and the crawl still runs. The clamp is visible, not silent.
2. `test_cli_crawl.py::test_delay_at_or_above_the_floor_is_passed_through` — `--delay 1.0` → factory `1.0`, no warning (floor inclusive); `--delay 2.5` → factory `2.5`, no warning; no rounding to whole seconds.
3. `test_cli_crawl.py::test_verbose_prints_one_line_per_page` — `-v` with the DEV fake → stdout contains, in order, `page=1/2 avail=4 check=5 not=7 req=1` and `page=2/2 avail=0 check=1 not=15 req=2`, exactly (no prefix, colour or trailing spaces), plus 02-14's summary line; nothing for the stub request, which is not a page.
4. `test_cli_crawl.py::test_without_verbose_only_the_summary_is_printed` — no `-v` → stdout has exactly one non-empty line and the per-page text appears nowhere; verbosity is opt-in.

Expected red symptom: `assert 0.2 == 1.0` for case 1 (the flag is passed through today) and `assert "page=1/2 avail=4 check=5 not=7 req=1" in ""` for case 3.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_cli_crawl.py -q`.

## Implement (green)
- `--delay` is resolved once, next to the other flag mapping: `effective = max(1.0, max(0.0, value))`, with the floor constant imported from `fetch.py` when it is exported there and shared otherwise — never a second independent `1.0` literal. A requested value below the floor prints one stderr warning naming the requested and the effective value, then continues with the effective one; at or above the floor, no warning.
- `-v` turns on INFO records for the `connex_watch` logger with a handler that writes the bare message to stdout and no formatter noise; without `-v` no INFO record reaches the user. The configuration is idempotent: two `main()` calls in one process must not double every line and must not leave a handler pointing at a closed stream.
- `crawl.py` emits one INFO record per parsed page from the page's own data: `page=<pageNo>/<totalPages> avail=<a> check=<c> not=<n> req=<requestCount so far>`, keys in that order, single spaces. Emit after the page is accepted (post-guard) and before the stop decision, so the page that triggers the stop is still reported. No record for the stub.
- No other log record changes level; 02-07's lying-pager warning stays a WARNING.
- One-sentence comment on the clamp and on the per-page record stating the format.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_cli_crawl.py -q` — seventeen green.

## Refactor / keep it DRY
The floor constant and the page-line format each exist once; the CLI resolves the delay in the same function that builds the policy. Re-run file and suite.

## Verification (how to test it)
```text
. .venv/bin/activate && pytest pipeline/tests/test_cli_crawl.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 17 passed, the pipeline suite green, `ruff` exits 0. Manual check: `. .venv/bin/activate && connex-watch crawl --preset dev --dry-run --delay 0.2` warns about the clamp and still prints the plan (no request); a second invocation in the same shell does not repeat lines twice.

## Out of scope (do not do)
- A `--quiet` flag, log files, timestamps in the per-page line, colour, progress bars, `tqdm`.
- Changing the summary line, the exit codes or the manifest — 02-14 / 02-12.
- Clamping `--max-pages`, validating `--out`, adding a `--user-agent` flag.
- Logging every card, retry or save: one line per page is the whole contract.
- Reworking `fetch.py`'s `min_interval` floor (02-01) or its retry logging.

## Definition of done
- [ ] All four cases exist, named as written, and pass; the thirteen earlier CLI cases still pass.
- [ ] Red run observed and quoted in the commit body.
- [ ] Target tests + `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `cli.py`, `crawl.py`, `test_cli_crawl.py` and this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit
```text
02-15: clamp the crawl delay and add per-page verbose lines

- points: 0.5
- red: test_delay_below_the_floor_is_clamped_with_a_warning failed (assert 0.2 == 1.0)
- clamp --delay to the 1.0 s floor and warn on stderr
- log one page=<n>/<total> avail/check/not/req line per parsed page under -v
- suite: 85 passed
```

Files in the commit: `pipeline/src/connex_watch/cli.py`, `pipeline/src/connex_watch/crawl.py`, `pipeline/tests/test_cli_crawl.py`, and this task file — nothing else.

## Traps and review notes
- Do not validate with argparse `type=` and raise: a small delay is corrected and reported, not rejected — the operator asked for a crawl.
- Do not put the clamp only in `fetch.py` and swallow the warning; the message belongs at the CLI boundary on stderr.
- Do not let the verbose handler survive the call or be added per request: duplicate lines are the classic symptom, and the tests call `main` several times in one process.
- Do not derive per-page counts from cumulative totals; each line reports its own page.
- Reviewer: check the exact strings in case 3 and that `req` is requests so far, not the page index.

## Plan samples are untrusted
`plan/stage-02-crawler.md` shows `page=7/546 avail=16 check=0 not=0 req=7` as an illustration; this task pins the DEV lines the fixtures actually produce. Verify the real counts in `search-dev-p1.html`/`search-dev-p2.html` before coding and record a deviation under `## Decisions` if the fixture disagrees.

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
