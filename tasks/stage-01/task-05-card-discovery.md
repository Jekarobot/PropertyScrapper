---
id: 01-05
stage: 01
title: Discover card nodes, positions and links
points: 2
owner: pipeline
status: todo
depends_on: [01-02, 00-05]
files:
  - pipeline/src/connex_watch/search_parser.py
  - pipeline/tests/test_search_parser_cards.py
---

# 01-05 · Discover card nodes, positions and links

**Points:** 2 · **Stage:** 01 — Search-page parser · **Owner lane:** pipeline · **Depends on:** 01-02, 00-05 · **Unblocks:** 01-06, 01-08, 01-09, 01-12, 01-15, 02-05

## Goal

Open a saved search page and return its result cards as an ordered list of scoped card handles, each carrying its 1-based position and its own detail link.

## Why this task exists

Everything the parser does afterwards happens on one card, so this is the only place that knows how a page is structured. Getting the boundary wrong — splitting on the `col-xl-3` wrapper, walking a card's siblings, or letting a selector reach into a neighbouring card — produces a parser that looks right on page 1 and quietly shuffles fields between units. Every later function receives exactly one card handle, and every error can name a position, which is the difference between "the site changed" and "page 1, card 7 is broken".

## Current state (preconditions)

- 00-05 is done: `pipeline/tests/conftest.py` loads the five frozen fixtures as strings (`search_dev_p1`, `search_dev_p2`, `search_boundary_p71`, `search_error_page`, `detail_cx84121`).
- 01-02 is done: `models.py` has `UnitCard`, `PageResult`, `SearchSpec`, `UnitState`.
- `pipeline/src/connex_watch/search_parser.py` and `ParseError` do not exist; `pytest pipeline -q` is green.

## Read before you start

- `CLAUDE.md` §3 (red must fail for the stated reason), §4 (2 points = 6–12 cases), §6, §7.
- `plan/01-discovery.md` §2 — the selector table and every quirk, especially quirk 5 (markup is not well-nested enough for `.parent` chains) and quirk 6 (`h2` also contains `Rent`).
- `plan/01-discovery.md` §3 — the end-of-data stub that has zero cards; §10 risk R1 (class renames).
- `plan/stage-01-search-parser.md` Task 1.2 — the original wording and its "do not split on the `col-xl-3` wrapper" trap; page assembly is sequenced into 01-17 (see `## Traps`).

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `pipeline/src/connex_watch/search_parser.py` | `ParseError` and `find_card_nodes(html)` |
| create | `pipeline/tests/test_search_parser_cards.py` | the eight cases below |

Nothing else may change: no field parser, no `parse_search_page`, no `PageResult` construction.

## Test first (red)

1. `test_dev_page_one_has_sixteen_cards` — `len(find_card_nodes(search_dev_p1)) == 16`.
2. `test_positions_are_one_based_and_in_document_order` — the positions are exactly 1…16, strictly increasing, no gaps, no duplicates.
3. `test_the_discovered_node_is_the_card_not_the_column_wrapper` — every handle's node carries the `condo-card` class and none carries `col-xl-3`, so a wrapper-based split fails here.
4. `test_card_link_is_the_parent_anchor_verbatim` — card 1's link is exactly `https://connex.in.th/en/detail/for-rent-condo-the-bangkok-sathorn-high-floor-river-view-bts-surasak-yannawa-sa-thon-bangkok-cx-84121`, character for character, with no rebuilt path and no trailing whitespace.
5. `test_all_search_fixtures_have_sixteen_cards` — `search_dev_p2` and `search_boundary_p71` also yield 16 handles, so discovery is not tuned to one page.
6. `test_the_error_stub_page_has_no_cards` — `search_error_page` yields `[]` and raises nothing; the 105-byte redirect body is not a parse failure.
7. `test_card_without_an_anchor_raises_a_located_parse_error` — delete the `href` attribute of the second card's anchor in `search_dev_p1` (string edit inside the test) → `ParseError` with `component == "card"`, `position == 2`, and a message containing `2`.
8. `test_each_handle_is_scoped_to_its_own_card` — on `search_boundary_p71` card 1's `.title_h2` reads `KnightsBridge Prime Sathorn` and card 2's reads `Equinox Phahol - Vibha`; a selector leaking across cards cannot produce both.

Expected red symptom: `ModuleNotFoundError: No module named 'connex_watch.search_parser'` for cases 1–6 and 8; case 7 needs `ParseError`, so it fails the same way first.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_cards.py -q` — eight red.

## Implement (green)

In `search_parser.py`:

- `ParseError(ValueError)` with `component: str` as its first positional argument and an optional `position: int | None = None`; `str(exc)` contains the component and, when given, the position. `ParseError("card", 2)` is this task's construction.
- `find_card_nodes(html: str) -> list[CardNode]` — parse the document once with BeautifulSoup's lxml parser, select every `.condo-card` in document order, and for each walk up to the nearest ancestor `a` element; the handle carries `position` (1-based), `href` (the anchor's `href`, only stripped of surrounding whitespace) and the node itself. A card whose anchor or `href` is missing raises `ParseError("card", position)` rather than being skipped. No cards at all returns an empty list — not an error.
- `CardNode` is a small immutable handle (frozen dataclass or `NamedTuple`) with documented field meanings; it stays parser-internal.
- One-sentence docstring on `find_card_nodes` and `ParseError` stating the contract.

Scoping notes that matter: every later selector is applied to `CardNode.node`, never to the document, because cards nest tables and headings inconsistently; the single permitted ancestor walk is the one to the wrapping anchor. lxml recovers broken markup without complaining, so the card count — not the parser's success — is the signal.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_cards.py -q` — eight green.

## Refactor / keep it DRY

The card selector and the anchor lookup exist once, here; no later task may re-select `.condo-card` from the full document. Delete any leftover helper that builds a second node list and re-run the suite.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_search_parser_cards.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Manual check: print the handle count and the first and last position for the three search fixtures — 16 handles, positions 1 and 16 — and an empty list for the stub page.

## Out of scope (do not do)

- `parse_search_page`, `PageResult` assembly, the pager, `is_error_stub`, `total_units` — 01-15, 01-16 and 01-17; this task must not construct a `UnitCard`.
- Any field parser: state (01-06), price (01-09), rooms and floor (01-10), size (01-11), identity (01-12), location (01-13), date (01-14).
- Network access, a fetcher, or reading a file inside the parser; logging skipped cards — there is no skipping.

## Definition of done

- [ ] All eight cases exist, named as written, and pass; the red run is quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `search_parser.py`, `test_search_parser_cards.py`, this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
01-05: discover search result cards

- points: 2
- red: test_dev_page_one_has_sixteen_cards failed (ModuleNotFoundError: search_parser)
- add find_card_nodes with 1-based positions, parent-anchor hrefs and a located ParseError
- note: no page assembly yet; parse_search_page lands in 01-17
- suite: 34 passed
```

Files in the commit: `search_parser.py`, `test_search_parser_cards.py`, this task file. Nothing else.

## Traps and review notes

- Do not split the HTML on `col-xl-3`: it is one wrapper per card today and a different class tomorrow. Select `.condo-card` and walk up to the anchor.
- Do not build `UnitCard` here: it requires every field and only 01-17 has all the field parsers; a partial card would force the model to be loosened.
- Case 7 edits the fixture string inside the test — do not commit a second HTML fixture. Reviewers: grep `find_parent` — it must appear exactly once (the anchor walk); any other ancestor chain is the quirk-5 bug.

## Plan samples are untrusted

`plan/stage-01-search-parser.md` Task 1.2 asks for `parse_search_page` in this same step. Its "select `.condo-card` and walk up to the `a[href]`" advice is right; the assembly is sequenced into 01-17 because the model cannot be satisfied earlier. Verify the BeautifulSoup API and parser choice against the installed versions, and record the sequencing deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
