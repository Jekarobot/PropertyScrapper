---
id: 01-06
stage: 01
title: Tri-state extraction and per-page histogram
points: 1
owner: pipeline
status: todo
depends_on: [01-05]
files:
  - pipeline/src/connex_watch/search_parser.py
  - pipeline/tests/test_search_parser_states.py
---

# 01-06 · Tri-state extraction and per-page histogram

**Points:** 1 · **Stage:** 01 — Search-page parser · **Owner lane:** pipeline · **Depends on:** 01-05 · **Unblocks:** 01-07

## Goal

Read each card's availability as one of the three `UnitState` members and count a page's cards into a histogram that always reports all three buckets.

## Why this task exists

Availability is only a CSS class plus a label on the search card — the site cannot filter by it, so this parse is the single source of the project's central claim. A page's histogram is also the cheapest early signal that the site changed: page 1 of the FULL query is all available today, and the moment it is not, the run should log a different shape. Counting must be one implementation, because the crawler's stop rule, the dataset's exclusion statistics and the nightly log all quote the same numbers.

## Current state (preconditions)

- 01-05 is done: `search_parser.py` has `ParseError` and `find_card_nodes`, which returns 16 handles per search fixture and none for the stub.
- 01-01 is done: `UnitState` and `from_label` are the only label mapping; `pytest pipeline -q` is green.
- `pipeline/tests/test_search_parser_states.py` does not exist.
- Fixture facts: DEV page 1 is 4 available / 5 check / 7 not available, DEV page 2 is 0 / 1 / 15, the FULL boundary page 71 is 12 available + 4 check and no not-available cards.

## Read before you start

- `CLAUDE.md` §3, §5 (one implementation per rule), §6.
- `plan/01-discovery.md` §2 — the state selectors (`available`, `not-available`, `check-availability`) and the state-label row; §5 — why a boolean would be wrong; §6 — the measured per-page shapes.
- `plan/02-architecture.md` §3.1 — the enum values and "only `available` is ever published".
- `plan/90-test-matrix.md` row P-05 — the requirement this task satisfies.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/search_parser.py` | `parse_state(card_node)` and `state_histogram(states)` |
| create | `pipeline/tests/test_search_parser_states.py` | the seven cases below |

Nothing else may change: no page assembly, no `PageResult`, no rules for a missing block (01-07).

## Test first (red)

1. `test_state_of_every_card_on_dev_page_one` — the 16 states in position order are the first four `available`, then positions 5–9 `check_availability`, then positions 10–16 `not_available`; assert the whole list, not a count.
2. `test_check_availability_comes_from_the_label` — position 5 reports `UnitState.CHECK_AVAILABILITY` (string value `check_availability`), whose label reads `Check Availability` while the CSS class is hyphenated: the label mapping decides the value, not the class name.
3. `test_histogram_of_dev_page_one` — exactly `{available: 4, check_availability: 5, not_available: 7}`. Matrix row P-05.
4. `test_histogram_of_dev_page_two` — exactly `{available: 0, check_availability: 1, not_available: 15}`; the zero bucket is present, not missing.
5. `test_histogram_of_boundary_page_71` — exactly `{available: 12, check_availability: 4, not_available: 0}` — the mixed page that makes the crawler's empty-streak rule necessary.
6. `test_histogram_has_three_keys_even_for_an_empty_page` — an empty state list yields three keys, all 0, in enum order; a page with no cards must not produce an empty mapping.
7. `test_states_are_enum_members_not_strings` — across all three fixtures every state is a `UnitState` member and equals its plain string, so the raw-page JSON carries `available`, never `UnitState.AVAILABLE`.

Expected red symptom: `ImportError: cannot import name 'parse_state' from 'connex_watch.search_parser'` for cases 1, 2 and 7, and the same for `state_histogram` in 3–6.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_states.py -q` — seven red.

## Implement (green)

In `search_parser.py`:

- `parse_state(card_node: CardNode) -> UnitState` — inside the handle's node, locate `.status-listing`, then the state element carrying one of the three documented state classes, then the `<span>` inside it; pass that span's text to `UnitState.from_label`. The class names only *find* the element: there must be no class→enum table anywhere, because `from_label` is the single label mapping (01-01).
- `state_histogram(states: Iterable[UnitState]) -> dict[UnitState, int]` — count into a dictionary initialised to zero for the three members in enum order, so a caller never gets a `KeyError` for an absent bucket. One sentence of docstring stating that.
- Both are pure: no I/O, no clock, no logging; selectors stay scoped to the handle.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_states.py -q` — seven green.

## Refactor / keep it DRY

The histogram is the only counting implementation: 02-07's stop rule and 03-05's exclusions call it or reuse its mapping, never re-count in a loop. `parse_state` must contain no literal `available`/`not_available` comparison — only the class names it needs. Re-run the suite after the refactor.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_search_parser_states.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Manual check: print the three histograms — 4/5/7, 0/1/15 and 12/4/0 in the order available / check_availability / not_available, with every key present.

## Out of scope (do not do)

- What happens when a card has no state block or an unknown label: the unguarded lookup stays, and 01-07 turns that crash into a located error.
- `is_error_stub`, pager parsing, `total_units`, `PageResult` — 01-08, 01-15, 01-16, 01-17.
- Publishing, filtering by state, or "available first" sorting — stage 03; a `state_counts` property on the model — stage 02 composes the mapping.
- Any state read from a detail page or from the card image `alt` text.

## Definition of done

- [ ] All seven cases exist, named as written, and pass; the red run is quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `search_parser.py`, `test_search_parser_states.py`, this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
01-06: extract tri-state availability

- points: 1
- red: test_state_of_every_card_on_dev_page_one failed (ImportError: parse_state)
- add parse_state via the state label and state_histogram with all three buckets
- histograms pinned on dev p1 (4/5/7), dev p2 (0/1/15) and boundary p71 (12/4/0)
- suite: 41 passed
```

Files in the commit: `search_parser.py`, `test_search_parser_states.py`, this task file. Nothing else.

## Traps and review notes

- Do not key the enum off the class name: the class is `check-availability` with a hyphen and the value is `check_availability` — a second mapping is exactly the drift 01-01 forbids.
- Never treat `check_availability` as available, not even in a helper named `is_available`.
- Do not return a bare `Counter`: a missing key would then be a `KeyError` at the call site instead of 0.
- The fixture cards carry an empty `<p>` beside the state div; a `select_one("p")` scoped too widely grabs it and reads nothing.
- Reviewers: grep for the three class names — they belong only to the element lookup — and for any numeric state count outside the parser.

## Plan samples are untrusted

`plan/stage-01-search-parser.md` Task 1.2 lists the histogram values and offers a choice between raising and returning a `None` state; this stage takes the raising route but the guard belongs to 01-07, so do not import the sample's pseudo-code. Verify every selector against the installed BeautifulSoup version and the frozen fixture, and record deviations under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
