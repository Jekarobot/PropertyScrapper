---
id: 01-08
stage: 01
title: Detect the out-of-range error stub
points: 0.5
owner: pipeline
status: todo
depends_on: [01-05]
files:
  - pipeline/src/connex_watch/search_parser.py
  - pipeline/tests/test_search_parser_pagination.py
---

# 01-08 · Detect the out-of-range error stub

**Points:** 0.5 · **Stage:** 01 — Search-page parser · **Owner lane:** pipeline · **Depends on:** 01-05 · **Unblocks:** none (the `PageResult` wiring lands in 01-17)

## Goal

Recognise the tiny JavaScript-redirect response the site returns past the last page, so a page beyond the end is classified as end-of-data and never as a parse failure.

## Why this task exists

Requesting a page number beyond the last page returns HTTP 200 with a body that contains no cards at all. Without a recogniser that response is indistinguishable from a markup change that broke card discovery, so a healthy run would abort or a short dataset would be published — one hides the site's real end, the other claims the market collapsed. The predicate is also what makes "zero cards inside the range is an error" safe later (rule P2), because it can say "this is a stub" and "this page has no cards" as two different things.

## Current state (preconditions)

- 01-05 is done: `find_card_nodes` returns 16 handles for real search pages and `[]` for the stub.
- `ParseError` exists; no stub predicate exists.
- `pipeline/tests/test_search_parser_pagination.py` does not exist yet — it is created here and grows in 01-15 and 01-16. `pytest pipeline -q` is green.

## Read before you start

- `CLAUDE.md` §3, §6, §9.
- `plan/01-discovery.md` §3 — the end-of-data table: the 200-with-stub row, the "filter matches nothing" row (`1 Of 0`, a normal page), the "last page partial" row, and the detection note "body length < 2 KB and no `.condo-card`".
- `plan/02-architecture.md` §6 — rules P2 and P3: zero cards in range is an error, the stub is end-of-data.
- `plan/90-test-matrix.md` row P-08 — the requirement this task satisfies, and the fixture `search-error-out-of-range.html`: 105 bytes, a script redirect to `/error/th`, zero cards.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/search_parser.py` | `detect_error_stub(html)` |
| create | `pipeline/tests/test_search_parser_pagination.py` | the three cases below |

Nothing else may change: no page assembly, no `PageResult`, no pager parsing.

## Test first (red)

1. `test_error_stub_page_is_detected` — `detect_error_stub(search_error_page)` is `True` and does not raise, even though the document has no cards and no pager. Matrix row P-08.
2. `test_real_search_pages_are_not_error_stubs` — `False` for `search_dev_p1`, `search_dev_p2` and `search_boundary_p71`, parameterised; a predicate that looks only at the card count would pass this and still fail case 3.
3. `test_page_without_cards_and_without_a_redirect_is_not_a_stub` — delete every card block from `search_dev_p1` (string edit inside the test) → `False`; this is the broken-markup case stage 02 must turn into a hard error, so it must not be excused as end-of-data.

Expected red symptom: `ImportError: cannot import name 'detect_error_stub' from 'connex_watch.search_parser'` for all three cases.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_pagination.py -q` — three red.

## Implement (green)

Add `detect_error_stub(html: str) -> bool` to `search_parser.py`:

- parse the document once and return `True` only when both conditions hold: there is no card element, **and** the document contains a client-side redirect to a path under `/error/` (the stub's `top.location.href` assignment). Either condition alone is not enough;
- a document with cards is never a stub, whatever it redirects to;
- a cardless document with no such redirect is not a stub — the caller decides what that means;
- it never raises: it is a classifier, and it must be callable before the pager is required;
- do not decide from `<meta>` tags, byte lengths or HTTP status; the redirect target plus the absence of cards is the contract;
- one sentence of docstring stating exactly that; keep the card test inside `find_card_nodes` and call it rather than re-selecting `.condo-card`.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_pagination.py -q` — three green.

## Refactor / keep it DRY

The predicate reuses `find_card_nodes` for the card test and introduces no second definition of "this page has cards". The redirect matcher is a single pattern used only here. Re-run the suite after the refactor.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_search_parser_pagination.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Manual check: call the predicate on all four search fixtures — expect True, False, False, False and no exception on the stub.

## Out of scope (do not do)

- Setting `is_error_stub` on `PageResult`, or raising for a normal page with zero cards: this is the pure predicate; 01-17 wires it in and 02-07 owns the in-range error rule.
- Pager parsing, `page_no`, `total_pages`, `total_units` — 01-15 and 01-16.
- Following the redirect, resolving `/error/th`, classifying other status codes, or deciding from a byte length.

## Definition of done

- [ ] All three cases exist, named as written, and pass; the red run is quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `search_parser.py`, `test_search_parser_pagination.py`, this file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
01-08: detect the out-of-range error stub

- points: 0.5
- red: test_error_stub_page_is_detected failed (ImportError: detect_error_stub)
- add detect_error_stub: no cards plus a redirect to /error/ is end-of-data
- a cardless page without the redirect stays a caller problem, not a stub
- suite: 47 passed
```

Files in the commit: `search_parser.py`, `test_search_parser_pagination.py`, this task file. Nothing else.

## Traps and review notes

- Do not use "fewer than 2 KB and no cards" as the rule: a filter that matches nothing returns a normal page with `1 Of 0`, and a future markup change could shrink a real page under any byte threshold.
- Do not return True just because the word `error` appears in a script or a link: anchor on the redirect target.
- Do not raise `ParseError` from the predicate — callers must be able to ask the question at any time.
- Reviewers: grep for a second `.condo-card` selection; the predicate must reuse the discovery function so the two definitions cannot drift.

## Plan samples are untrusted

`plan/stage-01-search-parser.md` Task 1.2 asserts the stub through `parse_search_page(...).is_error_stub`, which cannot exist before page assembly in 01-17; this task pins the predicate and 01-17 adds the integration assertion. Verify the sample against the installed parser and the frozen fixture, and record the sequencing deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
