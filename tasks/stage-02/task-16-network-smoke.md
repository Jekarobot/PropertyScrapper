---
id: 02-16
stage: 02
title: Opt-in live smoke test
points: 0.5
owner: pipeline
status: todo
depends_on: [02-14]
files:
  - pipeline/tests/test_live_smoke.py
---

# 02-16 · Opt-in live smoke test

**Points:** 0.5 · **Stage:** 02 — Crawler · **Owner lane:** pipeline · **Depends on:** 02-14 · **Unblocks:** nothing (stage review)

## Goal
Add the single test allowed to touch the live site: one request to DEV page 1 through the real fetcher, parsed by the real parser, asserted to still yield cards, the right page number and all three state keys — and skipped unless explicitly selected with `-m network`.

## Why this task exists
Fixtures freeze the site as it was on 2026-11-01; they cannot say that the markup changed yesterday, which is risk R1 — the failure that silently empties the dataset. One request per manual run is a cheap canary, and making it opt-in keeps the other 85 tests offline, deterministic and fast. It is also the only place where the politeness defaults meet the real server, so it doubles as a check that the UA and client settings are acceptable.

## Current state (preconditions)
- 02-01 to 02-15 are done: `ThrottledFetcher`, `crawl`, `store`, `cli` and the presets exist; the offline suite is green (85 tests, no skip yet — this task adds one).
- The `network` marker is registered in `pipeline/pyproject.toml` (00-03); `make check` exists (00-06) and is expected to run pytest with `-m "not network"`.
- `pipeline/tests/test_live_smoke.py` does **not** exist.
- `CLAUDE.md` §10: no live crawl without the owner's approval; this task adds no automated live run.

## Read before you start
- `CLAUDE.md` §3 (the only network test is marked and opt-in), §10 (one request, no detail pages), §11 (run pytest from the repo root).
- `plan/01-discovery.md` §8 — the budget: sequential, 1.2 s apart, identifying UA.
- `plan/stage-02-crawler.md` Task 2.5 — the three assertions and the 30 s bound; `plan/90-test-matrix.md` row P-24.
- `plan/assets/fixtures/README.md` — what the frozen DEV page contained, so a failure can be told apart from a site change.
- `pipeline/tests/conftest.py`, `pipeline/pyproject.toml` (marker), `Makefile` (check target).

## Files to touch
| Action | Path | What goes there |
|---|---|---|
| create | `pipeline/tests/test_live_smoke.py` | one marked test plus the skip guard |

Nothing else — in particular **not** the Makefile, `pyproject.toml`, `crawl.py`, `fetch.py` or the fixtures. No re-capture, no new fixture, no redaction step.

## Test first (red)
One case is the contract: the whole point is that the default suite has no second network test.

1. `test_live_smoke.py::test_dev_page_one_still_parses` — module-level `pytestmark = pytest.mark.network`; resolve the DEV preset URL through the existing preset loader (do not retype the URL), build page 1 with `build_search_url(spec, page_no=1)`, fetch it once with a real `ThrottledFetcher()` using every default (no injected transport, no injected sleep), parse with `parse_search_page`, and assert `len(cards) >= 1`, `page_no == 1`, the `state_histogram` over the page's card states has exactly the three buckets (all present, values may be zero), and fetch-plus-parse elapsed under 30 seconds measured with a monotonic clock. Do not assert `total_pages`, card counts, prices or unit ids — the ranking moves.

Skip guard: an autouse fixture in the module skips unless the active `-m` expression is exactly `network` (`request.config.getoption("-m") == "network"`), with the message `live site test: run pytest pipeline -m network`. A substring test is wrong: `-m "not network"` contains `network` and would leak a live request into `make check`.

Expected red symptom: without the marker and guard the test runs under a plain `pytest pipeline` and the suite reaches the internet — the failure this task removes, not a code failure. Once written, `pytest pipeline -q` shows exactly one `skipped`.

Run: `. .venv/bin/activate && pytest pipeline -m network -q` (with the owner's approval — one real request) and `. .venv/bin/activate && pytest pipeline -q` (must skip it).

## Implement (green)
- The module carries the marker and the autouse guard; both are required, because a marker alone only filters and does not protect a bare `pytest`.
- Import public entry points only: the preset loader from `cli` (or `tomllib` on `pipeline/presets.toml` if the loader is private — then keep one copy of the path logic), `SearchSpec`, `ThrottledFetcher`, `parse_search_page`. No `httpx` import and no client of our own.
- One fetch, one parse, no retry loop of ours, no crawl, no store, no writes to disk. Do not save the page and do not log the body (`CLAUDE.md` §10).
- The 30 s bound is a sanity timeout, not a performance assertion: assert it once on the measured value, with a message naming the elapsed seconds.
- No `sleep`, no warm-up request, no cookie handling.
- One-sentence docstring on the test stating what it proves and how to opt in.

Run: `. .venv/bin/activate && pytest pipeline -m network -q` — 1 passed; then `. .venv/bin/activate && pytest pipeline -q` — green with exactly one skipped.

## Refactor / keep it DRY
The preset path and URL come from one loader, never a second copy of the URL string; if that loader is private, use `tomllib` and a single module-level path constant rather than editing `cli.py`. The skip rule lives in this module only. Re-run both commands after refactoring.

## Verification (how to test it)
```text
. .venv/bin/activate && pytest pipeline/tests/test_live_smoke.py -m network -q
. .venv/bin/activate && pytest pipeline -q
. .venv/bin/activate && pytest pipeline -m network -q
ruff check pipeline
```

Expected: `1 passed` for the first and third commands (one real request each — run them only with the owner's approval), and the second green with exactly one skipped. Manual check: `make check` stays offline; if the Makefile does not pass `-m "not network"`, the guard still keeps it offline — report the gap, do not edit the Makefile (the filter belongs to 08-06).

## Out of scope (do not do)
- A second network test, a smoke test for the FULL query, page 71 or a detail page (04-09 owns the first detail fetch).
- Saving, re-capturing or redacting a fixture; committing any HTML.
- Making the default suite depend on the network, retrying the live request, asserting timing beyond the 30 s bound.
- Editing the Makefile, `pyproject.toml` markers or `conftest.py` global skip logic.
- A nightly or scheduled live run — `CLAUDE.md` §10 forbids unattended jobs; 08-05 only documents systemd units.
- Asserting prices, unit ids, `total_pages` or exact card counts.

## Definition of done
- [ ] `test_dev_page_one_still_parses` exists, is marked `network`, and is skipped by a plain run.
- [ ] `pytest pipeline -m network -q` passes when run with the owner's approval; the elapsed time is recorded in `## Result`.
- [ ] `pytest pipeline -q` green with exactly one skipped; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `test_live_smoke.py` and this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit
```text
02-16: add the opt-in live smoke test

- points: 0.5
- red: no failing case — the task adds a guard, not production code; the default run must stay offline
- add one network-marked test over the DEV page with an exact -m skip guard
- suite: 85 passed, 1 skipped
```

Files in the commit: `pipeline/tests/test_live_smoke.py` and this task file — nothing else.

## Traps and review notes
- The guard must be an exact `-m` comparison. `"network" in markexpr` lets `-m "not network"` through and puts a live request inside `make check`.
- Do not use `skipif(not os.environ.get(...))`: the documented opt-in is `-m network`, and an env var becomes a second, invisible switch.
- Do not assert `total_pages == 2`: the DEV project's inventory changes; the page number and the presence of cards are the contract.
- Do not let the real fetcher's default sleep fire: one request means no inter-request wait; if the test takes ~1.2 s longer than the request, something issued a second request.
- Reviewer greps the module for a hardcoded URL, `httpx`, `time.sleep` and `skipif` — none belong there.

## Plan samples are untrusted
`plan/stage-02-crawler.md` Task 2.5 names the assertions; the repo's `PageResult` field names and the preset loader's location win. Check both before coding and record a deviation under `## Decisions` if the state-count field is named differently.

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
