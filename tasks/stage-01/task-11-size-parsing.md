---
id: 01-11
stage: 01
title: Size in sqm, decimals, missing-size error
points: 1
owner: pipeline
status: todo
depends_on: [01-10]
files:
  - pipeline/src/connex_watch/search_parser.py
  - pipeline/tests/test_search_parser_fields.py
---

# 01-11 · Size in sqm, decimals, missing-size error

**Points:** 1 · **Stage:** 01 — Search-page parser · **Owner lane:** pipeline · **Depends on:** 01-10 · **Unblocks:** 01-17

## Goal

Read a card's area in square metres, keeping whole numbers integral and accepting decimals, and fail loudly when a card has no area at all.

## Why this task exists

Size is one of the two filter dimensions in the product (the range is inclusive on both ends), so a missing or wrong area corrupts the whole results list rather than one row. The size cell is also the trap of the card: unlike the three room icons it has an **empty** `alt`, so any icon-based match silently grabs another cell and returns the floor or the bathroom count — it must be identified by its `sqm.` caption. With bounds of 45–150 m² a card without a size cannot be placed in or out of range honestly, which is why the contract is an error rather than `None`.

## Current state (preconditions)

- 01-10 is done: the cell-by-icon helper and the three room/floor parsers exist; `test_search_parser_fields.py` has twelve green cases.
- `UnitCard.size_sqm` is `int | float` with a `> 0` bound (01-02); nothing fills it yet; `pytest pipeline -q` is green.
- Fixture facts: DEV card #1 is 65 sqm; boundary card #14's size cell reads `" <span>64</span> sqm."` with a leading space and an empty icon `alt`.

## Read before you start

- `CLAUDE.md` §3, §5, §6.
- `plan/01-discovery.md` §2 — the size row ("the `td` whose `p` text ends with `sqm.` → `span`") and quirk 4 (the size icon's empty `alt`).
- `plan/02-architecture.md` §3.2 — `sizeSqm` in the raw-page JSON; §5.2 rule F1 — size filters are inclusive, so the stored number must be exact.
- `plan/stage-01-search-parser.md` Task 1.3 — decimals accepted and missing size is a `ParseError` because size is a filter dimension.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/search_parser.py` | `parse_size_sqm(card_node)` |
| modify | `pipeline/tests/test_search_parser_fields.py` | the six cases below |

Nothing else may change: no `UnitCard` field change, no size formatting, no card assembly.

## Test first (red)

1. `test_size_of_the_first_card_is_65` — `search_dev_p1` position 1 → `65`, and the value is an `int`, not `65.0`.
2. `test_boundary_size_cell_with_a_leading_space_parses` — `search_boundary_p71` position 14 → `64`; that cell's text begins with a space before the number and ends with `sqm.`, and its icon `alt` is empty.
3. `test_decimal_size_is_accepted` — replace the first card's size number with `55.5` (string edit inside the test) → `55.5` as a float, while case 1 still returns an int.
4. `test_missing_size_cell_raises_a_located_parse_error` — delete the size cell (string edit) → `ParseError` with `component == "size"`, `position == 1`, message containing `size`; the function must not return `None`.
5. `test_every_card_has_an_empty_size_icon_alt_and_still_parses` — for all 48 cards the matched size cell's icon `alt` is the empty string and a size greater than zero is still returned; any icon-based implementation fails here.
6. `test_size_is_numeric_and_positive_on_every_fixture_card` — all 48 cards: the value is an `int` or a `float` (never a string, never `bool`) and is `> 0`.

Expected red symptom: `ImportError: cannot import name 'parse_size_sqm' from 'connex_watch.search_parser'` for cases 1–3, 5 and 6; case 4 fails the same way before it can assert the `ParseError`.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_fields.py -q` — six red, twelve green.

## Implement (green)

Add `parse_size_sqm(card_node: CardNode) -> int | float` to `search_parser.py`:

- choose the cell whose caption text, trimmed, ends with `sqm.` — the only reliable marker, because the icon `alt` is empty;
- take the first number in that cell and return an `int` when it has no fractional part, a `float` when it does (`55.5` stays `55.5`);
- the value must be strictly positive; a non-positive or unreadable number is an error;
- no size cell, an empty caption or no digits → `ParseError("size", position)`, message naming the component;
- one sentence of docstring: cell selection rule, numeric type rule, error condition.

No unit conversion, no rounding, no `m²` formatting, no model change. Do not reuse 01-10's icon helper: the size cell has no icon label, and pretending it does is exactly quirk 4.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_fields.py -q` — eighteen green.

## Refactor / keep it DRY

The "caption ends with `sqm.`" rule exists once, here, and this is the only place that decides int-versus-float for an area. Keep it separate from 01-10's first-integer scan even when both would parse a cell — they answer different questions. Re-run the suite after the refactor.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_search_parser_fields.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Manual check: print the size of all 16 cards of page 1 — every value between 45 and 150, all integers on this fixture, each matching the fixture's size column.

## Out of scope (do not do)

- Unit conversion (sq ft, wah), guessing the unit from `propertytype`, rounding or "fixing" an implausible area, or range validation against `minsize`/`maxsize` — the search filter does that server-side.
- Falling back to `None` or 0 for a missing size, reading from the card image `alt` text, a `size_sqm` model change (01-02), or the size-range filter itself (stage 03).

## Definition of done

- [ ] All six cases exist, named as written, and pass; the red run is quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `search_parser.py`, `test_search_parser_fields.py`, this file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
01-11: parse size in square metres

- points: 1
- red: test_size_of_the_first_card_is_65 failed (ImportError: parse_size_sqm)
- match the cell by its sqm. caption because the icon alt is empty
- keep ints integral, accept decimals, raise ParseError("size", position) when absent
- suite: 65 passed
```

Files in the commit: `search_parser.py`, `test_search_parser_fields.py`, this task file. Nothing else.

## Traps and review notes

- The size icon's `alt` is empty — reusing 01-10's helper selects the first cell with an empty alt, which is the size cell today and may not be tomorrow.
- `int("65.5")` raises and `float("65")` returns `65.0`; decide the type from the text, not from a post-hoc comparison.
- Do not strip `sqm.` with a pattern that also matches `sqm` inside a building name.
- Do not treat a missing size as out-of-range: the card would vanish from the dataset without any error.
- Reviewers: grep for an `alt`-based size lookup, `round(` and `float(` in this function.

## Plan samples are untrusted

`plan/stage-01-search-parser.md` Task 1.3 says "the `td` whose text ends with `sqm.`" and warns the icon `alt` is empty, but it names the boundary fixture's last card as the 64 example while that value is on position 14. Trust the fixture, not the sample's card index, and record the correction under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
