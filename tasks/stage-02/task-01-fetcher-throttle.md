---
id: 02-01
stage: 02
title: Fetcher protocol, throttle and politeness floor
points: 1
owner: pipeline
status: todo
depends_on: [01-01]
files:
  - pipeline/src/connex_watch/fetch.py
  - pipeline/tests/test_fetch.py
---

# 02-01 · Fetcher protocol, throttle and politeness floor

**Points:** 1 · **Stage:** 02 — Crawler · **Owner lane:** pipeline · **Depends on:** 01-01 · **Unblocks:** 02-02, 02-03, 04-09

## Goal
Create the single door to the network: a `Fetcher` protocol with `get(url) -> str` and a `ThrottledFetcher` that spaces request starts at least 1.2 s apart (plus 0–0.3 s jitter), refuses to be configured below the 1.0 s politeness floor, and returns the page body as text.

## Why this task exists
Every network call in the project — this crawl and stage 04's detail fetch — must pass through one module, or the politeness budget in `plan/01-discovery.md` §8 stops being enforceable. This task creates that module and the three injection seams (`sleep`, `clock`, `transport`) that make throttling testable without real time or sockets. Retries and refusals stay out: they are 02-02 and 02-03, built on this seam.

## Current state (preconditions)
- 01-01 is done: `models.py` exports `UnitState`, and `pytest pipeline -q` is green.
- `pipeline/src/connex_watch/fetch.py` and `pipeline/tests/test_fetch.py` do **not** exist.
- `httpx` is already pinned in `pipeline/pyproject.toml` (00-03); check the installed version and its `MockTransport` signature before relying on any API. No new dependency.
- The pytest marker `network` is registered (00-03); this task adds no network test.

## Read before you start
- `CLAUDE.md` §3 (determinism: inject the clock), §6 (untrusted samples), §10 (politeness).
- `plan/01-discovery.md` §8 — measured latency, the `1.2 s ± 0.3 s` budget, sequential requests.
- `plan/02-architecture.md` §6 rule P1 and §8 — one fetcher, the UA string, the `1.0 s` floor.
- `plan/stage-02-crawler.md` Task 2.1 and `plan/90-test-matrix.md` row P-11.

## Files to touch
| Action | Path | What goes there |
|---|---|---|
| create | `pipeline/src/connex_watch/fetch.py` | the protocol, the throttle, the client wiring |
| create | `pipeline/tests/test_fetch.py` | the six cases below, with fake `sleep`/`clock` |

Nothing else: no `models.py`, `__init__.py`, `pyproject.toml` or Makefile change.

## Test first (red)
Every case uses `httpx.MockTransport` and a fake `sleep` that records its argument; a fake `clock` returns a fixed value. No case may sleep for real, import `time.sleep` or open a socket.

1. `test_fetch.py::test_first_get_does_not_sleep` — one 200 `get()` → body returned and the recorded sleep list is exactly empty.
2. `test_fetch.py::test_second_get_sleeps_between_min_interval_and_jitter` — two `get()` calls → exactly one sleep, value `>= 1.2` and `<= 1.5`; pin the range, never a constant.
3. `test_fetch.py::test_min_interval_below_the_floor_raises_value_error` — `min_interval=0.2` raises `ValueError` mentioning `1.0` in the constructor; `min_interval=1.0` constructs (floor inclusive).
4. `test_fetch.py::test_get_returns_str_with_the_decoded_body` — UTF-8 bytes containing `CX-84121` → a `str` (not bytes) containing `CX-84121`.
5. `test_fetch.py::test_defaults_are_the_politeness_contract` — a default fetcher exposes read-only settings: `min_interval == 1.2`, `jitter == 0.3`, `timeout == 20.0`, `max_retries == 3`, `user_agent == "ConnexWatch/1.0 (+internal tool)"`.
6. `test_fetch.py::test_client_follows_redirects` — 302 with a `Location`, then 200 with a body → the final body is returned and the handler ran twice, proving `follow_redirects=True` rather than an internal attribute.

Expected red symptom: `ModuleNotFoundError: No module named 'connex_watch.fetch'`. A typo failure is not red.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_fetch.py -q` — six red for that reason.

## Implement (green)
- `Fetcher` — a `typing.Protocol` with one method, `get(url: str) -> str`; the crawl is typed against this, never against `ThrottledFetcher`.
- `ThrottledFetcher` — keyword-only constructor with the defaults above plus `sleep`, `clock` and an optional `transport` passed to the `httpx` client so tests inject `MockTransport`. Validate `min_interval` against the floor with `ValueError`; no other validation.
- `get()`: the first call never sleeps. From the second on, one sleep = remaining interval (measured with the injected `clock`) + a jitter draw in `[0, jitter]`. One sleep per attempt: 02-02 folds its backoff into the same value.
- One `httpx.Client` for the whole fetcher: `follow_redirects=True`, `timeout=timeout`, the UA header, the injected transport. Never a client per request.
- Return `response.content.decode("utf-8", errors="replace")` — always `str`, never a crash on a stray byte.
- No status-code branching here. One-sentence docstring on the protocol, the class and `get()`.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_fetch.py -q` — six green.

## Refactor / keep it DRY
The interval arithmetic and the single-sleep rule live in one private helper that 02-02 extends; the jitter draw, the UA literal and the floor constant exist once. Re-run the file after refactoring.

## Verification (how to test it)
```text
. .venv/bin/activate && pytest pipeline/tests/test_fetch.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 6 passed, then the whole pipeline suite green, `ruff` exits 0. Manual check: none — every case is offline; the only live request in this stage is 02-16.

## Out of scope (do not do)
- Retries, backoff, `Retry-After` (02-02); `403`/`404`, `FetchError`, `BlockedError` (02-03).
- The crawl loop, stop policy, parsing, checkpoints — 02-04 onwards.
- `--delay` clamping and CLI flags — 02-13 / 02-15; detail fetching and the coord cache — 04-09 / 04-05.
- Logging, retry counters, async, threads, cookies, sessions, proxies, pool tuning.

## Definition of done
- [ ] All six cases exist, named as written, and pass.
- [ ] Red run observed and quoted in the commit body.
- [ ] Target tests + `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `fetch.py`, `test_fetch.py` and this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit
```text
02-01: add throttled fetcher and politeness floor

- points: 1
- red: test_first_get_does_not_sleep failed (ModuleNotFoundError: No module named 'connex_watch.fetch')
- add Fetcher protocol and ThrottledFetcher with 1.2 s interval, 0.3 s jitter, 20 s timeout
- enforce the 1.0 s floor and inject sleep/clock/transport for offline tests
- suite: 11 passed
```

Files in the commit: `pipeline/src/connex_watch/fetch.py`, `pipeline/tests/test_fetch.py`, and this task file — nothing else.

## Traps and review notes
- Do not assert one exact sleep value for the throttle case: jitter makes it a range; a constant hides a jitter that was never applied.
- Do not use `time.time()` or `time.sleep()` inside `get()`: the injected `sleep`/`clock` are the only seams; a reviewer greps `time.sleep` and expects only the constructor default.
- Do not decode with `response.text` (header charset); the contract is forced UTF-8 with replacement.
- The UA is exactly `ConnexWatch/1.0 (+internal tool)`. `plan/02-architecture.md` §8 sketches a longer string with a contact address; invent no e-mail and no second form.
- Reviewer greps `import httpx` under `pipeline/src`: after 02-03 it appears only in `fetch.py`.

## Plan samples are untrusted
The constructor in `plan/stage-02-crawler.md` is illustrative: verify the installed `httpx` version and its `Client`/`MockTransport` signatures first, and record a deviation under `## Decisions` if the library disagrees. Never add a dependency because a sample mentions one.

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
