---
id: 02-03
stage: 02
title: 403 abort, 404, UTF-8 decoding, client settings
points: 1
owner: pipeline
status: todo
depends_on: [02-01]
files:
  - pipeline/src/connex_watch/fetch.py
  - pipeline/tests/test_fetch.py
---

# 02-03 · 403 abort, 404, UTF-8 decoding, client settings

**Points:** 1 · **Stage:** 02 — Crawler · **Owner lane:** pipeline · **Depends on:** 02-01 · **Unblocks:** 02-14

## Goal
Make refusals non-retryable and loud: a `403` raises `BlockedError` on the first response, a `404` raises `FetchError` without a retry, a body with invalid UTF-8 still returns a `str`, and the request carries the identifying UA.

## Why this task exists
The retry policy from 02-02 is only safe if the client can tell "busy" from "no". Retrying a `403` is how a polite crawler becomes a blocked one, and a `404` will never become a `200` within a run. One stray byte in a 169 KB page must not abort a 546-page crawl, so decoding has to be total. Finally this closes architecture rule P1 with an executable test: `fetch.py` is the only module allowed to import the HTTP library.

## Current state (preconditions)
- 02-01 and 02-02 are done: `Fetcher`, `ThrottledFetcher` and `FetchError` exist; retries cover `429`/`5xx`; `pytest pipeline/tests/test_fetch.py -q` is green (14 tests).
- `BlockedError` does not exist; `get()` currently returns the body of any non-retryable status.
- No module other than `fetch.py` imports `httpx` today; keep it that way.

## Read before you start
- `CLAUDE.md` §3 (one behaviour per test), §5 (P1 lives in one module), §6.
- `plan/01-discovery.md` §8 — Cloudflare front door; a block is an operational event, not a bug to retry through.
- `plan/02-architecture.md` §6 rule P1 and §8 — identify the crawler, stay read-only.
- `plan/stage-02-crawler.md` Task 2.1 cases 5, 7, 8 and the "only module importing `httpx`" note; `plan/90-test-matrix.md` P-11, P-21.

## Files to touch
| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/fetch.py` | `BlockedError`, the refusal branch, total decoding |
| modify | `pipeline/tests/test_fetch.py` | the six cases below |

Nothing else: no new file, no new dependency, no second module importing `httpx`.

## Test first (red)
Keep `min_interval=1.0, jitter=0.0` and the counting handler, so sleeps and request counts stay exact.

1. `test_fetch.py::test_403_raises_blocked_error_without_retry` — always `403` → exactly 1 request, no sleep, `BlockedError` whose message contains `403` and the URL and whose `.url` is the requested URL.
2. `test_fetch.py::test_404_raises_fetch_error_without_retry` — always `404` → exactly 1 request, no sleep, `FetchError` with `status == 404` and the URL; a `4xx` other than `429` is final.
3. `test_fetch.py::test_403_after_a_retry_still_aborts_immediately` — `503`, then `403` → exactly 2 requests, sleeps exactly `[1.0]` (the backoff already earned), then `BlockedError`; no third request.
4. `test_fetch.py::test_non_utf8_bytes_do_not_crash_the_decode` — the transport returns `Caf` plus the single byte `0xE9` → a `str` equal to `Caf` plus one replacement character (`U+FFFD`), no exception.
5. `test_fetch.py::test_requests_carry_the_identifying_user_agent` — the handler records `request.headers["user-agent"]` → exactly `ConnexWatch/1.0 (+internal tool)`; the header is on the wire, not only in the constructor.
6. `test_fetch.py::test_only_the_fetch_module_imports_httpx` — read every `*.py` under `pipeline/src/connex_watch/` → the set of files whose text contains an `httpx` import is exactly `{fetch.py}`; the test must keep passing as later stages add modules.

Expected red symptom: `ImportError: cannot import name 'BlockedError'` for case 1; for case 6 a failure listing the offending module or an empty set. Fix a typo failure before continuing.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_fetch.py -q`.

## Implement (green)
- `BlockedError` — one exception in `fetch.py`, separate from `FetchError` so a caller can catch a block without catching a transient failure; it carries `.url`, names `403` and the URL in its message, and is never retried.
- Status branch, in this order: `403` → `BlockedError` at once; `429`/`5xx` → the 02-02 retry path; any other `4xx` → `FetchError` at once with status and URL; `2xx` → the decoded body.
- Decoding is total: `response.content.decode("utf-8", errors="replace")` on every branch that returns a body; no `UnicodeDecodeError` may escape a crawl.
- The UA is sent on every attempt via the client's default header; the string comes from 02-01, never a second copy.
- `httpx` stays imported only in `fetch.py` — not even for a type hint elsewhere.
- One-sentence docstring on `BlockedError` stating who raises it and what the operator must do.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_fetch.py -q` — 20 green.

## Refactor / keep it DRY
The status-to-action decision is one small function (retryable / blocked / fatal / ok) used by the attempt loop; no scattered `if status ==` branches in `get()`. The UA literal and the "decode with replacement" rule each exist once. Re-run file and suite.

## Verification (how to test it)
```text
. .venv/bin/activate && pytest pipeline/tests/test_fetch.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 20 passed, the pipeline suite green, `ruff` exits 0. Manual check: `grep -rn "httpx" pipeline/src | grep -v fetch.py` prints nothing — the stage acceptance criterion, encoded by case 6.

## Out of scope (do not do)
- Authentication, cookies, `PHPSESSID`, proxies, per-host retry budgets.
- Special handling for `401`, `410`, `451` or any status the plan does not name; they take the generic `FetchError` branch.
- Backoff, jitter or throttle-floor changes — settled in 02-01 / 02-02.
- What the crawl does with a `BlockedError` (exit 3) — 02-05 and 02-14.
- Retrying connection errors, timeouts or TLS failures.

## Definition of done
- [ ] All six cases exist, named as written, and pass; the fourteen earlier `test_fetch.py` cases still pass.
- [ ] Red run observed and quoted in the commit body.
- [ ] Target tests + `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `fetch.py`, `test_fetch.py` and this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit
```text
02-03: abort on 403, fail fast on 404, decode UTF-8 safely

- points: 1
- red: test_403_raises_blocked_error_without_retry failed (ImportError: cannot import name 'BlockedError')
- add BlockedError for a 403 and stop retrying refusals
- make decoding total with errors="replace" and keep httpx inside fetch.py
- suite: 25 passed
```

Files in the commit: `pipeline/src/connex_watch/fetch.py`, `pipeline/tests/test_fetch.py`, and this task file — nothing else.

## Traps and review notes
- Do not subclass `BlockedError` from `FetchError` unless the crawl's `except` order is tested: a block and a transient failure have different operator responses.
- Do not retry a `404` "because it might be a cache miss" — the test counts requests.
- Do not drop bad bytes silently; `errors="replace"` preserves positions, so parser error positions stay meaningful.
- Do not import `httpx` elsewhere for typing; the protocol is enough.
- Reviewer: case 6 must read files, not a hardcoded list, so modules added in stages 03/04 are covered.

## Plan samples are untrusted
The plan names the errors but not their attributes; this task pins `.url` (and `.status` on `FetchError`) because 02-14 formats them. Verify the installed `httpx` behaviour for redirects and `4xx` and the `Response.content` type before coding; record a deviation under `## Decisions`.

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
