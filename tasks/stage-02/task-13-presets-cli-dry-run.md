---
id: 02-13
stage: 02
title: Presets, crawl CLI, dry-run plan
points: 2
owner: pipeline
status: todo
depends_on: [02-05, 02-09]
files:
  - pipeline/src/connex_watch/cli.py
  - pipeline/presets.toml
  - pipeline/tests/test_cli_crawl.py
---

# 02-13 · Presets, crawl CLI, dry-run plan

**Points:** 2 · **Stage:** 02 — Crawler · **Owner lane:** pipeline · **Depends on:** 02-05, 02-09 · **Unblocks:** 02-14, 02-15, 08-01

## Goal
Give the pipeline a real entry point: a `presets.toml` with the two named searches and a `connex-watch crawl` subcommand that resolves a preset or a URL, plans the pages, and either prints the plan (`--dry-run`) or crawls into a run directory — with the fetcher injected through `cli._make_fetcher` so tests never touch the network.

## Why this task exists
Until now the crawler was a library: two presets and a command make it usable, repeatable and scriptable, and they are the contract 08-01 calls from the nightly script. The CLI is also where operator choices (`--max-pages`, `--full`, `--resume`, `--delay`, `--out`, `--run-id`) become a `StopPolicy` and a run directory, so this is the one place that translates flags into behaviour — nothing downstream re-reads the command line.

## Current state (preconditions)
- 02-05 and 02-09 are done and 02-06/02-07/02-11/02-12 are done: `crawl(spec, *, fetcher, policy, store, resume)`, `RunStore(root, clock=...)`, `create`/`open`/`finish` exist and are green.
- `pipeline/src/connex_watch/cli.py` does **not** exist, although `[project.scripts] connex-watch = "connex_watch.cli:main"` was registered in 00-03 — the console script is broken until this task.
- `pipeline/presets.toml` and `pipeline/tests/test_cli_crawl.py` do not exist.
- `tomllib` is in the Python 3.12 standard library; no new dependency is needed or allowed.

## Read before you start
- `CLAUDE.md` §3 (no network, no real time in tests), §5 (argparse only, no CLI framework), §6, §11 (run pytest from the repo root).
- `plan/01-discovery.md` §4 — the DEV and FULL URLs, canonical parameter order, `page_no` last.
- `plan/stage-02-crawler.md` Task 2.4 — the usage block, the presets TOML, cases 1–3.
- `plan/02-architecture.md` §2 (`pipeline/presets.toml` is the named-search file) and §6 rules P1–P4; `plan/90-test-matrix.md` row P-13.

## Files to touch
| Action | Path | What goes there |
|---|---|---|
| create | `pipeline/presets.toml` | `[presets.dev]` and `[presets.daily]`, one `url` each |
| create | `pipeline/src/connex_watch/cli.py` | `main(argv)`, the `crawl` subcommand, `_make_fetcher` |
| create | `pipeline/tests/test_cli_crawl.py` | the eight cases below plus a local fake fetcher |

Nothing else: no `pyproject.toml` (the entry point exists), no Makefile, no `README.md`.

## Test first (red)
Call `main([...])` in-process inside `pytest.raises(SystemExit)` and assert `.code` plus the files on disk — never a subprocess. Monkeypatch `cli._make_fetcher` to return the local fake, which maps a page number to a body, records URLs and raises `AssertionError` for an unknown page. Point `--out` at `tmp_path`.

1. `test_cli_crawl.py::test_presets_hold_the_documented_urls_and_round_trip` — `tomllib` on `pipeline/presets.toml` gives `presets.dev.url` and `presets.daily.url` exactly equal to the DEV and FULL URLs of `plan/01-discovery.md` §4 (`page_no=1` last, empty params present), and for both `build_search_url(parse_search_url(url), page_no=1) == url`; a reordered or missing parameter fails here.
2. `test_cli_crawl.py::test_dry_run_prints_the_plan_and_makes_no_request` — `crawl --preset dev --dry-run` → exit 0, stdout exactly one line equal to the DEV page-1 URL, fake calls `[]`, no run directory under `--out`.
3. `test_cli_crawl.py::test_dry_run_with_max_pages_lists_the_planned_pages` — `--dry-run --max-pages 3` → exit 0, three lines in order, each `build_search_url(spec, page_no=n)` for `n` in 1, 2, 3, no request.
4. `test_cli_crawl.py::test_preset_and_url_are_mutually_exclusive_and_required` — neither flag → exit 2; both → exit 2; argparse writes usage to stderr in both cases.
5. `test_cli_crawl.py::test_unknown_preset_exits_two_and_lists_the_presets` — `--preset nope` → exit 2; stderr contains `nope`, `dev`, `daily`; nothing is created on disk.
6. `test_cli_crawl.py::test_crawl_saves_pages_and_finishes_the_run` — `crawl --preset dev --out <tmp> --run-id run-a` with the DEV fake → exit 0; `manifest.json`, `pages/page-0001.json`, `pages/page-0002.json` exist and `page-0003.json` does not; `stopReason == "empty_streak"`, `finishedAt` not null; exactly one non-empty stdout line (02-14 pins its content).
7. `test_cli_crawl.py::test_url_option_accepts_an_arbitrary_search_url` — `--url <FULL page-1 URL> --run-id run-b` with a fake serving three pages → exit 0, first requested URL exactly the FULL page-1 URL, and the manifest's `spec` block equal to that URL's parameter block.
8. `test_cli_crawl.py::test_default_run_id_is_a_utc_timestamp` — a real crawl with no `--run-id` leaves exactly one directory under `--out` whose name matches `YYYYMMDDTHHMMSSZ` (UTC, no separators).

Expected red symptom: `ModuleNotFoundError: No module named 'connex_watch.cli'` for every case except 1, which fails on the missing `presets.toml`. A typo failure is not red.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_cli_crawl.py -q`.

## Implement (green)
`presets.toml`: a `[presets.dev]` and a `[presets.daily]` table, each with one `url` key holding the URLs from `plan/01-discovery.md` §4 verbatim.

`cli.py`:
- `main(argv=None) -> int` — argparse, dispatch to the `crawl` handler, `SystemExit` with the documented code (argparse raises 2 for usage errors; the handler raises `SystemExit(0)` or 02-14's codes). `argv=None` means `sys.argv[1:]`, so the console script works.
- Flags exactly as planned: `--preset {dev,daily}` **or** `--url URL` (mutually exclusive, required), `--out DIR` (default `data/raw`), `--run-id ID`, `--max-pages N`, `--full`, `--delay SECONDS` (default `1.2`), `--resume`, `--dry-run`, `-v`.
- Presets are read with `tomllib` from `pipeline/presets.toml` located relative to the package file (never the CWD) and parsed with `parse_search_url`; an unknown name → exit 2 plus a stderr line listing the available names.
- Flag mapping in one place: `--max-pages`/`--full` build the `StopPolicy` (`scan_all=--full`); `--out` is the store root; `--run-id` defaults to the UTC start time as `YYYYMMDDTHHMMSSZ`; `--resume` opens the existing run (`RunStore.open`) and creates when the directory is absent; `--delay` goes to the fetcher factory.
- `_make_fetcher(min_interval)` is the only place a `ThrottledFetcher` is constructed, so tests monkeypatch that name; the CLI never imports `httpx`.
- `--dry-run` constructs no fetcher and makes no request: one URL per line for pages `1..N`, `N` being `--max-pages` when given and `1` otherwise, built with `build_search_url(spec, page_no=n)`; nothing else on stdout.
- The success summary is one line; 02-14 owns its exact fields, so print the data it needs (pages, cards, per-state counts, stop reason, duplicates, elapsed) without inventing a second format.
- One-sentence docstring on `main`, the handler and `_make_fetcher`.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_cli_crawl.py -q` — ten green.

## Refactor / keep it DRY
Preset loading, spec resolution (`--preset` or `--url`), policy construction and store opening are each one function; the handler only orchestrates. Page URLs come from `build_search_url` — the CLI never assembles query strings. Re-run file and suite.

## Verification (how to test it)
```text
. .venv/bin/activate && pytest pipeline/tests/test_cli_crawl.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 8 passed, the pipeline suite green, `ruff` exits 0. Manual check: `. .venv/bin/activate && connex-watch crawl --preset dev --dry-run` prints one URL and exits 0; `git status --short` shows no `data/` entry after the suite.

## Out of scope (do not do)
- Exit codes 3 and 4, operator messages, the exact summary format — 02-14.
- The `--delay` clamp and `-v` per-page lines — 02-15.
- `build`, `check`, `publish`, `nightly` — 03-13, 03-14, 08-02, 08-06.
- A config file for delay/UA, environment variables, `--version`, shell completion, colours.
- Reading run history to guess `total_pages` in a dry-run (see the trap). Touching `README.md` — the exit-code table lands in 08-11.

## Definition of done
- [ ] All eight cases exist, named as written, and pass.
- [ ] Red run observed and quoted in the commit body.
- [ ] Target tests + `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `presets.toml`, `cli.py`, `test_cli_crawl.py` and this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit
```text
02-13: add presets and the crawl subcommand

- points: 2
- red: test_crawl_saves_pages_and_finishes_the_run failed (ModuleNotFoundError: No module named 'connex_watch.cli')
- add presets.toml with the dev and daily URLs
- add main/crawl with policy, store, resume and dry-run wiring
- suite: 76 passed
```

Files in the commit: `pipeline/presets.toml`, `pipeline/src/connex_watch/cli.py`, `pipeline/tests/test_cli_crawl.py`, and this task file — nothing else.

## Traps and review notes
- A dry-run cannot know `total_pages` without a request, so the plan's acceptance line "`--dry-run` lists exactly 2 URLs" is unreachable offline; this task prints `1..N` with `N` from `--max-pages` (default 1). Do not "fix" that by fetching page 1 in a dry-run.
- Do not let the handler build URLs or page numbers; `build_search_url` is the only builder.
- Do not resolve `presets.toml` from the CWD: the console script must work from any directory.
- Do not import `httpx` in `cli.py`; the factory returns the protocol type. No CLI framework, no settings object, no second parser for tests.
- Reviewer: case 3 must assert both "fake never called" and "no run directory" — a dry-run that writes a manifest passes a stdout-only assertion.

## Plan samples are untrusted
`plan/stage-02-crawler.md` shows the TOML and the usage block; copy the URLs from `plan/01-discovery.md` §4 (the TOML there is illustrative) and check the repo's `SearchSpec` signature before coding. Record any deviation under `## Decisions`.

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
