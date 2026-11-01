---
id: 02-02
stage: 02
title: Retries, exponential backoff, Retry-After
points: 2
owner: pipeline
status: todo
depends_on: [02-01]
files:
  - pipeline/src/connex_watch/fetch.py
  - pipeline/tests/test_fetch.py
---

# 02-02 · Retries, exponential backoff, Retry-After

**Points:** 2 · **Stage:** 02 — Crawler · **Owner lane:** pipeline · **Depends on:** 02-01 · **Unblocks:** 02-03

## Goal
Make one `get()` survive a transient failure: retry `429` and `5xx` up to three attempts, wait exponentially (`1 s`, then `2 s`) or exactly as long as `Retry-After` asks, and then raise a single `FetchError` carrying the status and the URL.

## Why this task exists
The site sits behind Cloudflare and occasionally answers `503` or `429` under load (`plan/01-discovery.md` §8). A 546-page crawl that dies on the first `503` is useless; one that retries without waiting gets the IP blocked. Bounded attempts, growing waits and honoured `Retry-After` are part of the politeness contract, and one typed failure is what lets the CLI print an operator message instead of a traceback. Everything is proven with an injected `sleep`, so the suite never waits.

## Current state (preconditions)
- 02-01 is done: `fetch.py` exports `Fetcher` and `ThrottledFetcher` with the `sleep`/`clock`/`transport` seams; `pytest pipeline/tests/test_fetch.py -q` is green (6 tests).
- The constructor already accepts and stores `max_retries` (default `3`); nothing reads it yet.
- `FetchError` does **not** exist. `test_fetch.py` exists and is extended, not replaced.

## Read before you start
- `CLAUDE.md` §3 (red first, determinism, no weakened assertions), §6.
- `plan/01-discovery.md` §8 — backoff on `429`/`503`, abort after three consecutive failures.
- `plan/02-architecture.md` §6 rule P1 and §8 — backoff defaults are non-negotiable.
- `plan/stage-02-crawler.md` Task 2.1 cases 3 and 4; `plan/90-test-matrix.md` P-11.

## Files to touch
| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/fetch.py` | the retry loop, backoff, `Retry-After`, `FetchError` |
| modify | `pipeline/tests/test_fetch.py` | the eight cases below, all with a fake `sleep` |

Nothing else: no `models.py`, no `crawl.py` (it does not exist yet), no `pyproject.toml`.

## Test first (red)
Build the fetcher with `min_interval=1.0, jitter=0.0` (legal: floor value, zero jitter) so the recorded sleeps are exact numbers. The transport handler counts its calls and can return a scripted status sequence.

1. `test_fetch.py::test_503_is_retried_three_times_then_raises_fetch_error` — always `503` → exactly 3 requests, recorded sleeps exactly `[1.0, 2.0]` (none after the final failure), `FetchError` whose message contains `503` and the URL and whose `status == 503`, `url` equal to the request URL.
2. `test_fetch.py::test_503_then_200_returns_the_body` — `503`, then `200` with `CX-84121` → that body as `str`, 2 requests, sleeps exactly `[1.0]`.
3. `test_fetch.py::test_429_with_retry_after_sleeps_exactly_five` — `429` with `Retry-After: 5`, then `200` → sleeps exactly `[5.0]`, 2 requests; the header wins over the backoff.
4. `test_fetch.py::test_429_without_retry_after_uses_backoff` — `429` with no header, then `200` → sleeps exactly `[1.0]`; a missing header is not a free retry.
5. `test_fetch.py::test_retry_after_value_is_parsed_not_hardcoded` — `Retry-After: 2`, then `200` → sleeps exactly `[2.0]`.
6. `test_fetch.py::test_max_retries_one_makes_a_single_attempt` — `max_retries=1`, always `503` → exactly 1 request, no sleep, `FetchError`; the setting counts attempts, not retries.
7. `test_fetch.py::test_fetch_error_exposes_status_and_url` — as case 1, but assert `.status` and `.url` directly (the CLI formats them in 02-14).
8. `test_fetch.py::test_retry_waits_use_only_the_injected_sleep` — monkeypatch `time.sleep` to raise `AssertionError`, run case 1 → `FetchError` is still raised, proving no real delay.

Expected red symptom: `ImportError: cannot import name 'FetchError'`, or `assert 1 == 3` on the request count once the name exists. Fix a typo failure before continuing.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_fetch.py -q`.

## Implement (green)
- `FetchError` — one exception in `fetch.py`, constructor taking message, status and URL, exposing `.status` (`int | None`) and `.url` (`str`); the message contains both. It is the only failure type for a bad status in this task.
- Retry set: `429` and every `5xx`. A retryable status consumes one attempt; `max_retries` is the total number of attempts (`3` = one call plus two retries); exhaustion raises `FetchError` for the **last** status.
- Wait before attempt `n` (`n >= 2`): `2 ** (n - 2)` seconds — `1` before attempt 2, `2` before attempt 3. Never sleep after the final failure.
- A `Retry-After` header on a `429` replaces the backoff for that wait when it parses as whole seconds; an HTTP-date, junk or a negative value falls back to the backoff.
- One sleep per attempt, equal to the larger of the remaining throttle interval and the backoff/`Retry-After`, plus the jitter draw — extend 02-01's helper, do not add a second wait path.
- A retried request counts as one `get()` for the throttle clock; the decoded body is returned exactly as in 02-01. One-sentence docstring on `FetchError` and the retry helper.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_fetch.py -q` — all green, no real waits.

## Refactor / keep it DRY
The attempt loop owns every wait; `crawl.py` must never see a retry, a sleep or a status code. The backoff formula and the `Retry-After` parse exist once, reusing 02-01's throttle arithmetic. Re-run the file and the suite.

## Verification (how to test it)
```text
. .venv/bin/activate && pytest pipeline/tests/test_fetch.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 14 passed, the whole pipeline suite green, `ruff` exits 0. Manual check: none; a real `503` is only observable in the opt-in live test (02-16).

## Out of scope (do not do)
- `403`, `404`, `BlockedError`, UTF-8 hardening and client settings — 02-03.
- Retry counters, retry logging, HTTP-date `Retry-After`, jittered backoff, a cap beyond three attempts, circuit breakers.
- Retrying connection errors, DNS failures or timeouts; only status codes are specified.
- The crawl loop, checkpoints, CLI exit codes and `--delay` — 02-05, 02-09, 02-14.

## Definition of done
- [ ] All eight cases exist, named as written, and pass; the six 02-01 cases still pass.
- [ ] Red run observed and quoted in the commit body.
- [ ] Target tests + `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `fetch.py`, `test_fetch.py` and this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit
```text
02-02: retry transient failures with backoff and Retry-After

- points: 2
- red: test_503_is_retried_three_times_then_raises_fetch_error failed (ImportError: cannot import name 'FetchError')
- add FetchError carrying status and url
- retry 429 and 5xx up to three attempts with 1 s and 2 s waits; honour Retry-After
- suite: 19 passed
```

Files in the commit: `pipeline/src/connex_watch/fetch.py`, `pipeline/tests/test_fetch.py`, and this task file — nothing else.

## Traps and review notes
- `max_retries` counts attempts. Reading it as "extra retries" makes the always-503 case issue 4 requests; the assertion catches it — do not "fix" the test.
- Do not sleep after the last failure: the list must be `[1.0, 2.0]`, not `[1.0, 2.0, 4.0]`.
- Do not put the backoff in a second sleep next to the throttle sleep; one wait per attempt (the maximum) keeps the list exact and the politeness honest.
- Do not retry `403`/`404` "for symmetry" — 02-03 asserts a single request for both.
- Reviewer: grep the test file for `time.sleep` and for `1.2` in the retry cases; neither belongs there.

## Plan samples are untrusted
`plan/stage-02-crawler.md` writes `2^n` without a base case; this task pins `1 s` then `2 s` in the tests. Verify the installed `httpx` API and the existing `fetch.py` before using any snippet and record a deviation under `## Decisions`.

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
