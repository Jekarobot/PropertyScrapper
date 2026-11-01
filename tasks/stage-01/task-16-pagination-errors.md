---
id: 01-16
stage: 01
title: Missing pager error; total units derived, not SEO
points: 0.5
owner: pipeline
status: todo
depends_on: [01-15]
files:
  - pipeline/src/connex_watch/search_parser.py
  - pipeline/tests/test_search_parser_pagination.py
---

# 01-16 · Missing pager error; total units derived, not SEO

**Points:** 0.5 · **Stage:** 01 — Search-page parser · **Owner lane:** pipeline · **Depends on:** 01-15 · **Unblocks:** 01-17

## Goal

Turn a page whose pager is gone into a named error, and derive the total unit count from the pager and the card count instead of the page's SEO metadata.

## Why this task exists

A page with no pager means the markup changed, and a crawl that silently loses the total page count would either stop after page 1 or run past the end; both are invisible in a dataset that still looks plausible. The unit total is the other half: the `<meta description>` prints a number that looks authoritative and is not — it is comma-formatted SEO copy, it drifts, and the discovery doc lists it among the things that look like data but are not. Deriving the number from `total_pages` and the current card count keeps one arithmetic rule that the raw page can state and the reviewer can re-check.

## Current state (preconditions)

- 01-15 is done: `parse_pager(html)` returns `(page_no, total_pages)` and the pagination test file has seven green cases.
- No guard for a missing pager exists; nothing derives a unit total.
- `PageResult.total_units` exists as `int | None` (01-02) and is filled by 01-17.
- Fixture facts: DEV page 1 is `1 Of 2` with 16 cards; the boundary page is `71 Of 546` with 16 cards and its meta description reads `Fine 8,734 Condo For Rent …`.

## Read before you start

- `CLAUDE.md` §3, §6, §9.
- `plan/01-discovery.md` §3 — the end-of-data table and "do not depend on the pager to iterate"; §9 — the `<meta name="description">` row: SEO copy, only sometimes in sync with the pager.
- `plan/02-architecture.md` §3.2 — `totalUnits` in the raw page JSON; §6 rules P2–P4 — pagination drives stopping, not a stored estimate.
- `plan/stage-01-search-parser.md` Task 1.5 item 5 — "`total_units` is derived … never from the SEO `<meta description>`".
- The measured totals in `plan/01-discovery.md` §6: FULL = 8734 units over 546 pages, DEV = 32 over 2 pages.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/search_parser.py` | the missing-pager guard and `derive_total_units(page_no, total_pages, card_count)` |
| modify | `pipeline/tests/test_search_parser_pagination.py` | the five cases below |

Nothing else may change: no `PageResult` assembly, no stop policy, no meta parsing.

## Test first (red)

1. `test_missing_pager_raises_a_pagination_error` — remove the `number-page` class from the boundary fixture (string edit inside the test) → `ParseError` with `component == "pagination"`, `position is None`, and a message containing `pagination`.
2. `test_total_units_on_dev_page_one` — `derive_total_units(1, 2, 16)` → `32`, which is the measured DEV total.
3. `test_total_units_on_a_last_partial_page` — `derive_total_units(546, 546, 14)` → `8734`; the last page of the FULL query is partial, and this value matches the measured total.
4. `test_total_units_on_an_interior_page_is_an_upper_bound` — `derive_total_units(71, 546, 16)` → `8736`, which is deliberately not the `8,734` printed in that page's meta description.
5. `test_derived_total_ignores_the_meta_description` — the boundary fixture's meta description says `Fine 8,734 Condo For Rent` while `derive_total_units(71, 546, 16)` is `8736`, and the parser module's source contains no `meta` lookup; assert both, so a future "helpful" meta read fails the suite.

Expected red symptom: `ImportError: cannot import name 'derive_total_units' from 'connex_watch.search_parser'` for cases 2–5; case 1 fails with `TypeError`/`AttributeError` because no guard exists yet.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_pagination.py -q` — five red, seven green.

## Implement (green)

In `search_parser.py`:

- guard `parse_pager`: when the pager container or its `N Of M` text is missing, raise `ParseError("pagination")` with no position — the failure is page-level, not card-level. Reuse the container lookup from 01-15 rather than re-selecting it.
- add `derive_total_units(page_no: int, total_pages: int, card_count: int) -> int` with one documented page-size constant of 16: when `page_no == total_pages` return `(total_pages - 1) * 16 + card_count`, otherwise return `total_pages * 16`. Document that the interior-page value is an upper bound because the last page may be partial.
- reject nonsense input (any argument below 1) with `ValueError`; do not clamp silently.
- the function takes numbers only — it must not receive HTML, a soup object, or a document, which is the structural proof that it cannot read `<meta>`.
- one sentence of docstring per item stating the rule and the bound.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_pagination.py -q` — twelve green.

## Refactor / keep it DRY

The page-size constant and the arithmetic exist once, here; 01-17 calls the function with the real card count. The pager container lookup is shared with 01-15. Do not add a second "total units" path in the page builder. Re-run the suite after the refactor.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_search_parser_pagination.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Manual check: `derive_total_units(71, 546, 16)` prints 8736; print the boundary page's meta description and see 8,734 — the two numbers must differ, proving the SEO value was not used.

## Out of scope (do not do)

- Reading `<meta>` in any form, including as a cross-check or a warning; `PageResult` assembly, `is_error_stub` wiring or the card builder — 01-17.
- The stop policy, `empty_streak`, `--max-pages` or the crawl loop — stage 02 (02-05, 02-06).
- Raising the pagination error for the error-stub page: the stub is classified first (01-08, wired in 01-17).
- Reconciling the derived total with the dataset's unit count — stage 03.

## Definition of done

- [ ] All five cases exist, named as written, and pass; the red run is quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `search_parser.py`, `test_search_parser_pagination.py`, this file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
01-16: guard the pager and derive total units

- points: 0.5
- red: test_missing_pager_raises_a_pagination_error failed (AttributeError: NoneType)
- raise ParseError("pagination") when the pager is gone
- derive_total_units from pager and card count; 16 cards per page, last page partial
- suite: 90 passed
```

Files in the commit: `search_parser.py`, `test_search_parser_pagination.py`, this task file. Nothing else.

## Traps and review notes

- The stub page has no pager either; if the guard runs before stub detection, every out-of-range page becomes a pagination error. The caller (01-17) must classify the stub first, and `parse_pager` must stay a plain function so that order is visible.
- Do not "fix" the upper bound by reading the meta description, a sitemap, or the last pager link: the estimate is allowed to be approximate, a second source is not.
- Do not accept `card_count > 16` silently; it means two pages were concatenated.
- Reviewers: grep the module for `meta` — there must be no match — and confirm `derive_total_units` has no document parameter.

## Plan samples are untrusted

`plan/stage-01-search-parser.md` Task 1.5 writes the estimate as "`16·(n−1)+k` as a hint only" without pinning when that form applies. This task pins the two branches explicitly and treats the interior-page value as an upper bound. Verify the arithmetic against the measured totals in `plan/01-discovery.md` §6 and record the refinement under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
