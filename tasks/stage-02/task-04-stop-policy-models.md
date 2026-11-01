---
id: 02-04
stage: 02
title: StopPolicy and CrawlResult models
points: 0.5
owner: pipeline
status: todo
depends_on: [01-02]
files:
  - pipeline/src/connex_watch/models.py
  - pipeline/tests/test_crawl_stop.py
---

# 02-04 · StopPolicy and CrawlResult models

**Points:** 0.5 · **Stage:** 02 — Crawler · **Owner lane:** pipeline · **Depends on:** 01-02 · **Unblocks:** 02-05, 02-09

## Goal
Add the two value objects the crawl speaks in: an immutable `StopPolicy` with the documented defaults and a `CrawlResult` that carries cards, pages, the state histogram, the stop reason, the request count, the partial flag and the duplicate count.

## Why this task exists
The crawl loop is the first place where several rules meet — when to stop, what was seen, whether the run is complete — and those rules must be nameable before they are implemented. Fixing the defaults here (`max_pages=None`, `empty_streak=2`, `scan_all=False`) means 02-05 to 02-13 cannot invent their own stopping behaviour, and fixing `CrawlResult` now gives the CLI (02-13) and the manifest (02-09/02-12) one shape to read.

## Current state (preconditions)
- 01-02 is done: `models.py` defines `UnitState`, `UnitCard`, `PageResult` and `SearchSpec` as pydantic models; `pytest pipeline -q` is green.
- `StopPolicy` and `CrawlResult` do **not** exist; `pipeline/tests/test_crawl_stop.py` does **not** exist (this task creates it, 02-05 to 02-08 extend it).
- `pipeline/src/connex_watch/crawl.py` does **not** exist (02-05 creates it).

## Read before you start
- `CLAUDE.md` §3 (determinism), §5 (a rule has one implementation), §6.
- `plan/02-architecture.md` §3.1 (domain objects) and §3.2 (manifest fields `pagesFetched`, `stopReason`, `requestCount`).
- `plan/02-architecture.md` §6 rule P4 — the stop policy this model encodes.
- `plan/stage-02-crawler.md` Task 2.2 (the `StopPolicy` sample and the `CrawlResult` field list); `plan/90-test-matrix.md` row P-10.

## Files to touch
| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/models.py` | `StopPolicy` and `CrawlResult` only |
| create | `pipeline/tests/test_crawl_stop.py` | the four cases below |

Nothing else: no crawl loop, no store, no CLI, no change to `UnitCard`/`PageResult`/`SearchSpec`.

## Test first (red)
1. `test_crawl_stop.py::test_stop_policy_defaults_and_immutability` — `StopPolicy()` has `max_pages is None`, `empty_streak == 2`, `scan_all is False`; assigning to `empty_streak` raises (`dataclasses.FrozenInstanceError` or the model's immutability error — pin the concrete type, never a bare `Exception`).
2. `test_crawl_stop.py::test_stop_policy_accepts_explicit_overrides` — `StopPolicy(max_pages=2, empty_streak=1, scan_all=True)` reports those values; two policies built from the same arguments compare equal; `max_pages=0` is stored as given (validation is the caller's job).
3. `test_crawl_stop.py::test_crawl_result_defaults_for_a_fresh_walk` — `CrawlResult()` has `cards == []`, `pages == []`, `state_histogram` with exactly the keys `available`, `check_availability`, `not_available` all `0`, `stop_reason is None`, `request_count == 0`, `partial is False`, `duplicate_units == 0`.
4. `test_crawl_stop.py::test_crawl_result_holds_the_final_counts` — construct with two cards, one page, a histogram `{available: 4, check_availability: 5, not_available: 7}`, `stop_reason="empty_streak"`, `request_count=3`, `partial=True`, `duplicate_units=2` → every value reads back unchanged and `stop_reason` is the plain string, not an enum member.

Expected red symptom: `ImportError: cannot import name 'StopPolicy' from 'connex_watch.models'`. A typo failure is not red.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_crawl_stop.py -q`.

## Implement (green)
- `StopPolicy` — an immutable dataclass with exactly `max_pages: int | None = None`, `empty_streak: int = 2`, `scan_all: bool = False`. No methods, no validation, no helper.
- `CrawlResult` — a pydantic model: `cards: list[UnitCard] = []`, `pages: list[PageResult] = []`, `state_histogram` defaulting to the three `UnitState` members mapped to `0` (a `str` enum, so comparing it to `{available: 4, ...}` holds), `stop_reason: str | None = None`, `request_count: int = 0`, `partial: bool = False`, `duplicate_units: int = 0`. Use factories so two results never share a list or dict.
- Field meanings, documented on the class because later tasks rely on them: `cards` — every card from every parsed page in page order, **duplicates included**; `pages` — the parsed `PageResult` objects in fetch order, `len(pages)` is the manifest's `pagesFetched` (the out-of-range stub is not a page); `request_count` — page requests issued by this crawl invocation (one per `get()`, retries invisible); `partial` — the walk stopped before covering its intended range; `duplicate_units` — cards whose id was already seen (extras, not distinct ids).
- `stop_reason` is a plain string; the vocabulary is `last_page`, `empty_streak`, `max_pages`, `error_stub`, `aborted`, where `aborted` is reserved for stage 08 and no stage-02 code writes it. No enum — the manifest stores the string.
- One-sentence docstring on both classes. No `httpx`, `crawl`, `store` or `cli` import here.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_crawl_stop.py -q` — four green.

## Refactor / keep it DRY
The three histogram keys come from `UnitState` members, not retyped literals; the stop-reason strings are documented in one place and referenced by name in 02-06. Re-run the suite after refactoring.

## Verification (how to test it)
```text
. .venv/bin/activate && pytest pipeline/tests/test_crawl_stop.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 4 passed, the pipeline suite green, `ruff` exits 0. Manual check: none — this task adds data only.

## Out of scope (do not do)
- The crawl loop, fetching, stop evaluation, streak counting — 02-05 / 02-06.
- Raw page and manifest JSON, atomic writes — 02-09 / 02-10 / 02-12.
- Validation of policy values (`max_pages=0`, negative streaks) — no test asks for it.
- An enum for `stop_reason`, a policy method that decides stopping, a `CrawlResult.add_page()` accumulator.
- Exports from `connex_watch/__init__.py`.

## Definition of done
- [ ] All four cases exist, named as written, and pass.
- [ ] Red run observed and quoted in the commit body (`ImportError` line).
- [ ] Target tests + `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `models.py`, `test_crawl_stop.py` and this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit
```text
02-04: add stop policy and crawl result models

- points: 0.5
- red: test_stop_policy_defaults_and_immutability failed (ImportError: cannot import name 'StopPolicy')
- add frozen StopPolicy with max_pages/empty_streak/scan_all defaults
- add CrawlResult with cards, pages, histogram, stop reason, request count, partial, duplicates
- suite: 29 passed
```

Files in the commit: `pipeline/src/connex_watch/models.py`, `pipeline/tests/test_crawl_stop.py`, and this task file — nothing else.

## Traps and review notes
- Two `CrawlResult()` instances must not share the default list or dict: the second test would pass while the crawl silently mixed two runs.
- Do not type `stop_reason` as an enum or a `Literal` that omits `aborted`; the manifest vocabulary is wider than what stage 02 writes.
- `pages` is a list of page objects, **not** a page count. Writing an int breaks 02-11 (resume rebuilds the result) and 02-12 (`pagesFetched`).
- Reviewer greps for `"available"` / `"check_availability"` literals outside `models.py`; the histogram keys must come from `UnitState`.

## Plan samples are untrusted
The plan's `StopPolicy` sample is a frozen dataclass while `models.py` is pydantic-based; both are fine only if they behave as the tests state. Check the installed pydantic version before relying on default-factory or immutability behaviour, and record a deviation under `## Decisions`.

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
