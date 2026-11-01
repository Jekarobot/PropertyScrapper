---
id: 01-10
stage: 01
title: Bedrooms, bathrooms, floor via icon labels
points: 1
owner: pipeline
status: todo
depends_on: [01-09]
files:
  - pipeline/src/connex_watch/search_parser.py
  - pipeline/tests/test_search_parser_fields.py
---

# 01-10 · Bedrooms, bathrooms, floor via icon labels

**Points:** 1 · **Stage:** 01 — Search-page parser · **Owner lane:** pipeline · **Depends on:** 01-09 · **Unblocks:** 01-11

## Goal

Read bedrooms, bathrooms and floor from the card's option table by matching each icon's `alt` label to its cell, returning `None` for the two counts a card may omit.

## Why this task exists

The option table is the densest part of the card: four cells in a fixed visual order, each with an icon, a caption and a number, and that order is not something the site promises. Matching by the icon label instead of by position keeps the parser stable when a card drops a cell, and it is what makes "the card does not state a floor" expressible as `None` rather than a bogus 0. Bedrooms feed the exact-match filter (studio is 0, not "no bedrooms"), while the floor is shown to the user and must never claim "floor 0" for a missing value.

## Current state (preconditions)

- 01-05 is done: card handles are scoped to one card and carry positions.
- 01-09 is done: `parse_price_thb` exists and `test_search_parser_fields.py` has five green cases.
- `UnitCard` requires `bedrooms` (`>= 0`) and allows `bathrooms`/`floor` to be `None` (01-02); nothing fills them yet. `pytest pipeline -q` is green.

## Read before you start

- `CLAUDE.md` §3, §5, §6.
- `plan/01-discovery.md` §2 — the bedroom/bathroom/floor rows (icon `alt` → closest `td` → `p span`) and quirk 5: do not rely on `td.parent` chains, scope selection to the card.
- `plan/02-architecture.md` §3.2 — the three integers, with `floor` allowed to be null in the dataset only through this path.
- `plan/stage-01-search-parser.md` Task 1.3 — the room and floor rows, including "missing `Floors` → `floor is None` (not 0)".
- Fixture facts: DEV card #1 is 1 bedroom / 1 bathroom / floor 27; boundary card #1 is 2/1/29; boundary card #15 is a 3-bedroom, 2-bathroom unit on floor 3.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/search_parser.py` | one cell-by-icon helper plus `parse_bedrooms`, `parse_bathrooms`, `parse_floor` |
| modify | `pipeline/tests/test_search_parser_fields.py` | the seven cases below |

Nothing else may change: no size parsing, no card assembly.

## Test first (red)

1. `test_first_card_room_counts_and_floor` — `search_dev_p1` position 1 → bedrooms 1, bathrooms 1, floor 27.
2. `test_boundary_card_room_counts` — `search_boundary_p71` position 1 → bedrooms 2, bathrooms 1, floor 29, so bedrooms is not hard-coded to 1.
3. `test_three_bedroom_unit_on_the_boundary_page` — position 15 of the same fixture → bedrooms 3, bathrooms 2, floor 3; the floor 3 must not be confused with a missing value.
4. `test_studio_keeps_zero_bedrooms` — replace the first card's bedroom number with 0 (string edit inside the test) → bedrooms `0`, not `None`.
5. `test_missing_floors_cell_returns_none` — delete the Floors cell from the first card (string edit) → `parse_floor` returns `None`, never `0`.
6. `test_missing_bathroom_cell_returns_none` — delete the Bathroom cell → `parse_bathrooms` returns `None`.
7. `test_the_icon_label_selects_the_cell_not_the_caption` — rewrite the first card's bedroom caption text to `Bathroom 3` while keeping its icon alt `Bedroom` → bedrooms 3; a caption-driven implementation returns 1 here.

Expected red symptom: `ImportError: cannot import name 'parse_bedrooms' from 'connex_watch.search_parser'` for cases 1–4 and 7, and the same for `parse_bathrooms`/`parse_floor` in 5–6.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_fields.py -q` — seven red, five green.

## Implement (green)

One private helper plus three thin public functions in `search_parser.py`:

- the helper takes the handle and an icon `alt` value and returns that cell by scanning the card's `td` elements and matching the `img` alt inside each cell. Do not walk parents: the discovery doc records that the nesting is not trustworthy for `.parent` chains, and one scan is easier to reason about;
- the first integer in the cell's caption text is the value; surrounding text is ignored;
- `parse_bedrooms(card_node) -> int` — a missing Bedroom cell raises `ParseError("bedrooms", position)`, because the model requires it and 0 is a legitimate value;
- `parse_bathrooms(card_node) -> int | None` and `parse_floor(card_node) -> int | None` — return `None` when the cell or its number is missing, never 0;
- one sentence of docstring per function: source cell, fallback, error.

The helper exists once for all three fields. No positional indexes (`td[0]`, `td[2]`), no title-casing, no unit conversion.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_fields.py -q` — twelve green.

## Refactor / keep it DRY

The three functions share the one cell-by-icon helper; no three selector loops. The "first integer in a text" rule is not generalised into a shared number parser, because 01-11's size rule is caption-ending-based and different. Re-run the suite after the refactor.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_search_parser_fields.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Manual check: print bedrooms/bathrooms/floor for all 16 cards of page 1 — the fixture reads 1/1/27, 1/1/9, 1/1/41, 1/1/26, … with no value 0 or empty.

## Out of scope (do not do)

- Size in sqm — 01-11 owns it, including the boundary fixture's size cell whose icon alt is empty.
- A building-level height, a "high floor" badge, any derived statistic, a missing cell read as 0, or the caption word parsed as a field (the icon alt is the key).

## Definition of done

- [ ] All seven cases exist, named as written, and pass; the red run is quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `search_parser.py`, `test_search_parser_fields.py`, this file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
01-10: parse bedrooms bathrooms and floor

- points: 1
- red: test_first_card_room_counts_and_floor failed (ImportError: parse_bedrooms)
- add a cell-by-icon helper and parse_bedrooms/parse_bathrooms/parse_floor
- missing floor and bathroom cells yield None; studio keeps 0 bedrooms
- suite: 59 passed
```

Files in the commit: `search_parser.py`, `test_search_parser_fields.py`, this task file. Nothing else.

## Traps and review notes

- Do not index the cells by position: the size cell's icon alt is empty, so a positional parser passes these fixtures and breaks on the first card that omits a cell.
- Do not use `img.find_parent("td")`; the discovery doc warns the nesting is unreliable — scan the card's cells.
- 0 and `None` are different: a studio has 0 bedrooms, a card without a floor has `None`; do not normalise either into the other. Reviewers: grep for `td[0]`, `td[1]`, `find_parent` and `or 0` — all are the bugs this task exists to prevent.

## Plan samples are untrusted

`plan/stage-01-search-parser.md` Task 1.3 says "icon `alt` → nearest `td` → first integer in its `p` text", but the discovery doc's quirk 5 contradicts the parent-walk part for broken markup; this task scans the card's cells instead. Verify any selector against the installed BeautifulSoup version and record the deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
