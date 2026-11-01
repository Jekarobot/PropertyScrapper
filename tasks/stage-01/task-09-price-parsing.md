---
id: 01-09
stage: 01
title: Parse price THB from the detail line
points: 0.5
owner: pipeline
status: todo
depends_on: [01-05]
files:
  - pipeline/src/connex_watch/search_parser.py
  - pipeline/tests/test_search_parser_fields.py
---

# 01-09 · Parse price THB from the detail line

**Points:** 0.5 · **Stage:** 01 — Search-page parser · **Owner lane:** pipeline · **Depends on:** 01-05 · **Unblocks:** 01-10

## Goal

Read a card's monthly rent as an integer number of THB from the price line, ignoring the currency symbol and any trailing label.

## Why this task exists

Price is the field a renter filters on and the number the budget query is built around, so it must be an exact integer, not a display string and not a float. The line is deceptively mixed: it starts with the `฿` symbol on every card, carries thousands separators, and on some pages is followed by an inline `Condo` label, so a naive conversion either fails or keeps the label. Once the value is an `int`, the `price_thb > 0` rule from 01-02 rejects the nonsense cards that a markup change would otherwise introduce.

## Current state (preconditions)

- 01-05 is done: `find_card_nodes` returns handles with positions and a scoped node.
- `UnitCard.price_thb` already requires an integer `> 0` (01-02); nothing fills it yet. `pipeline/tests/test_search_parser_fields.py` does not exist — it is created here and grows in 01-10 and 01-11, and `pytest pipeline -q` is green.

## Read before you start

- `CLAUDE.md` §3, §4 (0.5 points = one mechanical change plus a focused test set), §6.
- `plan/01-discovery.md` §2 — the price row (`.detail_1` text, digits only) and quirk 1 (the `฿` symbol and the inline `Condo` span).
- `plan/02-architecture.md` §3.2 — `priceThb` is an integer in the raw page JSON.
- `plan/stage-01-search-parser.md` Task 1.3 — the price row of the field table, and "never `float()` prices".

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/search_parser.py` | `parse_price_thb(card_node)` |
| create | `pipeline/tests/test_search_parser_fields.py` | the five cases below |

Nothing else may change: no other field, no card assembly, no model change.

## Test first (red)

1. `test_price_of_the_first_card_is_45000` — position 1 of `search_dev_p1` returns the integer `45000`; the text is `฿45,000`, so symbol and separator must both be gone.
2. `test_price_ignores_the_inline_condo_label` — position 1 of `search_dev_p2` returns `48000`; that line is followed by an inline `Condo` span, so a scan that also collects trailing letters or a second number fails here.
3. `test_price_strips_thousands_separators` — replace the first card's price text with `฿120,000` (string edit inside the test) → `120000`.
4. `test_price_is_an_int_on_every_fixture_card` — for all 48 cards of the three search fixtures the result is an `int` (not `float`, not `bool`, not a string) and is `> 0`.
5. `test_missing_price_line_raises_a_located_parse_error` — delete the first card's price block (string edit) → `ParseError` with `component == "price"`, `position == 1`, message containing `price`.

Expected red symptom: `ImportError: cannot import name 'parse_price_thb' from 'connex_watch.search_parser'` for cases 1–4; case 5 fails the same way before it can assert the `ParseError`.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_fields.py -q` — five red.

## Implement (green)

Add `parse_price_thb(card_node: CardNode) -> int` to `search_parser.py`:

- take the card's price element text (the `detail_1` block) — no other element may be consulted;
- take the first run of digits, allowing commas inside the number;
- drop the separators and convert to an integer: THB per month, never a float, never rounded, never a `Decimal`;
- ignore the currency symbol, the inline label and surrounding whitespace;
- when the element or the digit run is absent, raise `ParseError("price", position)` using the handle's position — the model's `> 0` bound is not a substitute, because a missing element would crash before validation;
- one sentence of docstring: text → integer THB → `ParseError` when unreadable.

Selectors stay scoped to the handle; no logging, no currency handling, no per-month annotation.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_fields.py -q` — five green.

## Refactor / keep it DRY

The digit-run pattern is the parser's only money scanner; 01-10 and 01-11 read small integers from a different structure and must not copy these rules. Do not build a shared "parse any number" helper until a second caller genuinely needs the same separators logic. Re-run the suite after the refactor.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_search_parser_fields.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Manual check: print the price of all 16 cards on page 1 — values between 35000 and 50000, all integers, matching the fixture's price column.

## Out of scope (do not do)

- Bedrooms, bathrooms, floor (01-10), size (01-11), currency conversion, VAT, deposit, or a price per square metre.
- Display formatting (`฿45,000` with a comma) — that is the web layer (05-12); sale prices — v1 is rent-only.
- Reading a price from the card image `alt` text or the detail page.

## Definition of done

- [ ] All five cases exist, named as written, and pass; the red run is quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `search_parser.py`, `test_search_parser_fields.py`, this file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
01-09: parse price thb from the card detail line

- points: 0.5
- red: test_price_of_the_first_card_is_45000 failed (ImportError: parse_price_thb)
- add parse_price_thb: digits only, commas removed, int never float
- raise ParseError("price", position) when the line or the number is missing
- suite: 52 passed
```

Files in the commit: `search_parser.py`, `test_search_parser_fields.py`, this task file. Nothing else.

## Traps and review notes

- Never call `float()` on a price and never store a `Decimal`: `45000.0` would serialise as `45000.0` in the raw page JSON and break that key's contract.
- Do not strip digits from the whole card text — `floor 27`, `65 sqm` and the unit id would all leak into the match.
- Do not treat the inline `Condo` label as a property-type signal; property type comes from the query.
- The `฿` character is multi-byte: a byte-level regex on the HTML is fine, slicing text by byte index is not.
- Reviewers: grep for `float(` and for a second `detail_1` selection — both must be absent.

## Plan samples are untrusted

`plan/stage-01-search-parser.md` Task 1.3 proposes a first-match digit pattern and warns against `float()`. The proposal is close to this contract but its exact regex is unverified; check it against the fixture's `฿45,000` and the `Condo` line before trusting it, and record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
