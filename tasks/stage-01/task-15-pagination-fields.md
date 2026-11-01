---
id: 01-15
stage: 01
title: Page number and total pages from the pager
points: 0.5
owner: pipeline
status: todo
depends_on: [01-05]
files:
  - pipeline/src/connex_watch/search_parser.py
  - pipeline/tests/test_search_parser_pagination.py
---

# 01-15 · Page number and total pages from the pager

**Points:** 0.5 · **Stage:** 01 — Search-page parser · **Owner lane:** pipeline · **Depends on:** 01-05 · **Unblocks:** 01-16

## Goal

Read the current page number and the total number of pages from the pager's own text line, never from the "next" link.

## Why this task exists

Two numbers decide when a crawl stops, and both are printed in one line: `N Of M`. The next link is a tempting shortcut and the wrong one — on page 1 it points at page 2, on the last page it disappears, and a parser that reads it reports the wrong current page and never recognises the end. Reading the text line is also what makes the boundary page special: it says `71 Of 546` while carrying a mix of available and unavailable cards, which is the fact the crawler's stop rule is built on.

## Current state (preconditions)

- 01-05 is done: `find_card_nodes` and `ParseError` exist; 01-08 created `test_search_parser_pagination.py` with three green stub cases.
- No pager parser exists; `pytest pipeline -q` is green.
- Fixture facts: DEV page 1 says `1 Of 2`, DEV page 2 says `2 Of 2`, the boundary page says `71 Of 546`. The text carries two trailing spaces.
- Verified markup fact: the `<p>` with `N Of M` is **not** inside `ul.number-page`; it sits beside it, inside the `div` that also carries the `numx` class. The discovery doc's selector table says `.number-page p`, which matches nothing in these fixtures.

## Read before you start

- `CLAUDE.md` §3, §6, §9.
- `plan/01-discovery.md` §3 — the pagination contract, the `/^\s*(\d+)\s+Of\s+(\d+)/` shape, "both numbers are 1-based", and "do not depend on the pager to iterate".
- `plan/02-architecture.md` §6 rule P4 — the stop policy uses `pageNo == totalPages`.
- `plan/90-test-matrix.md` row P-08 and the `search-boundary-p71.html` row of the fixtures README.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/search_parser.py` | `parse_pager(html)` |
| modify | `pipeline/tests/test_search_parser_pagination.py` | the four cases below |

Nothing else may change: no total-units derivation, no missing-pager error, no `PageResult`.

## Test first (red)

1. `test_dev_page_one_pager` — `parse_pager(search_dev_p1)` → `(1, 2)`; the same document's next link points at `page_no=2`, so returning `(2, 2)` fails here.
2. `test_dev_page_two_pager` — `(2, 2)`; on the last page the "next" arrow is disabled and carries no `href`, so a link-based parser has nothing to read.
3. `test_boundary_page_pager` — `(71, 546)` from the line with two trailing spaces; whitespace tolerance is part of the contract.
4. `test_pager_text_wins_over_the_next_link` — rewrite the pager text to `5 Of 9` in `search_boundary_p71` (string edit inside the test) while leaving the hrefs at `page_no=70`/`72` → `(5, 9)`; the two numbers come from the text, not from either link.

Expected red symptom: `ImportError: cannot import name 'parse_pager' from 'connex_watch.search_parser'` for all four cases.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_pagination.py -q` — four red, three green.

## Implement (green)

Add `parse_pager(html: str) -> tuple[int, int]` to `search_parser.py`:

- locate the pager container by its `ul.number-page` list, then read the `p` element in that list's parent — not inside the list, because the fixtures place the text beside it;
- match the trimmed text against the `N Of M` shape and return `(page_no, total_pages)` as two `int`s;
- both numbers are 1-based and neither may be zero; require `page_no >= 1` and `total_pages >= 1`;
- ignore the list items, the active marker and every `href` on the page;
- one sentence of docstring: container, text shape, returned pair.

This task defines the happy path only. A document without the pager is 01-16's error case, so do not add that guard here.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_pagination.py -q` — seven green.

## Refactor / keep it DRY

The `N Of M` pattern and the container lookup exist once, here; 01-16 reuses the lookup for its error case instead of re-selecting the container. No helper that parses `page_no` out of a link — that would reintroduce the bug this task removes. Re-run the suite after the refactor.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_search_parser_pagination.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Manual check: print the pair for the three search fixtures — `(1, 2)`, `(2, 2)`, `(71, 546)` — and confirm the boundary page reports 71 even though its next href says 72.

## Out of scope (do not do)

- Missing pager error, `total_units`, and any page-level model — 01-16 and 01-17.
- Building the next URL, walking pages, or deciding the stop reason — stage 02 (02-05, 02-06).
- Parsing `page_no` from a link, from the active list item, or from the request URL.
- Validating that `page_no <= total_pages` here; that comparison belongs to the stop policy.

## Definition of done

- [ ] All four cases exist, named as written, and pass; the red run is quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `search_parser.py`, `test_search_parser_pagination.py`, this file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
01-15: parse page number and total pages

- points: 0.5
- red: test_dev_page_one_pager failed (ImportError: parse_pager)
- add parse_pager reading the N Of M text beside ul.number-page
- whitespace tolerated; the numbers never come from the next link
- suite: 85 passed
```

Files in the commit: `search_parser.py`, `test_search_parser_pagination.py`, this task file. Nothing else.

## Traps and review notes

- The `.number-page p` selector from the discovery doc's table matches nothing in the frozen fixtures — the `p` is a sibling of the `ul`. Select from the container instead, and record the correction in `## Decisions`.
- Do not take the current page from the `active` list item: on the boundary page the active item is a bare `<li class='active'><a>71</a></li>`, but on a page whose window of numbers has been rewritten it may lag.
- Do not accept `0 Of 0` as valid: the site prints `1 Of 0` for an empty result set, and a zero page number would break the stop policy's arithmetic.
- Reviewers: grep for `page_no=` and `href` inside `parse_pager` — neither may appear.

## Plan samples are untrusted

`plan/stage-01-search-parser.md` Task 1.5 and `plan/01-discovery.md` §3 agree on the text shape but the table's selector is wrong for the frozen markup; that contradiction is resolved in favour of the fixture. Verify any regex against the installed Python `re` and record the deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
