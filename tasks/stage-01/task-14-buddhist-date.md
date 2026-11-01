---
id: 01-14
stage: 01
title: Buddhist-era date to ISO, malformed dates
points: 0.5
owner: pipeline
status: todo
depends_on: [01-12]
files:
  - pipeline/src/connex_watch/search_parser.py
  - pipeline/tests/test_search_parser_identity.py
---

# 01-14 · Buddhist-era date to ISO, malformed dates

**Points:** 0.5 · **Stage:** 01 — Search-page parser · **Owner lane:** pipeline · **Depends on:** 01-12 · **Unblocks:** 01-17

## Goal

Convert the card's "Updated" date from the Thai Buddhist calendar to a plain ISO date, and return `None` for a date that cannot be read.

## Why this task exists

The site prints dates in the Buddhist era, so `29/10/2569` is really 2026-10-29; storing it raw would place every listing 543 years in the future and any "recently updated" comparison would be nonsense. The date is also only a "listing edited" marker, not an availability signal, so it must never decide state. A malformed or missing date is not worth failing a page for — the field is decorative for the product — which is why this one field degrades to `None` while size and state raise.

## Current state (preconditions)

- 01-12 is done: identity and image parsers exist with six green cases in `test_search_parser_identity.py`.
- `UnitCard.updated_on` is a `date | None` (01-02) and is required to be passed explicitly; nothing fills it yet.
- `pytest pipeline -q` is green; no date parser exists.
- Fixture facts: card #1's update line reads `Updated 29/10/2569`; the boundary page's cards carry other Buddhist-era dates in the same shape.

## Read before you start

- `CLAUDE.md` §3 (one behaviour per test), §4 (0.5 points = a small change and a focused test set), §6.
- `plan/01-discovery.md` §2 — the updated row (`.status-update span`, `Updated 29/10/2569`, subtract 543) and quirk 3.
- `plan/01-discovery.md` §9 — "Updated" means listing edited, not available since; state comes only from the card's state block.
- `plan/02-architecture.md` §3.2 — `updatedOn` is an ISO date string in the raw page JSON.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/search_parser.py` | `parse_updated_on(card_node)` |
| modify | `pipeline/tests/test_search_parser_identity.py` | the four cases below |

Nothing else may change: no timezone handling, no "days ago" helper, no card assembly.

## Test first (red)

1. `test_buddhist_date_is_converted_to_iso` — `search_dev_p1` position 1 → `date(2026, 10, 29)` from `Updated 29/10/2569`; a naive parse returns year 2569 and fails here.
2. `test_another_buddhist_year_is_converted` — replace the first card's update text with `Updated 01/01/2500` (string edit inside the test) → `date(1957, 1, 1)`; pins that the rule is "subtract 543", not "subtract 543 only for 2569".
3. `test_malformed_date_returns_none_without_raising` — replace the text with `Updated n/a` → `None`, and specifically no exception; a parser that raises here would make a display-only field break a whole page.
4. `test_missing_update_line_returns_none` — delete the `status-update` block from the first card (string edit) → `None`.

Expected red symptom: `ImportError: cannot import name 'parse_updated_on' from 'connex_watch.search_parser'` for all four cases.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_identity.py -q` — four red, twelve green.

## Implement (green)

Add `parse_updated_on(card_node: CardNode) -> date | None` to `search_parser.py`:

- read the update element's text, strip the `Updated` label and whitespace, and accept only a `DD/MM/YYYY` day-first numeric date;
- subtract 543 from the year and return a `datetime.date` — never a `datetime`, never a string, never a timestamp;
- if the element is missing, the text has no date shape, or the date is not a valid calendar date (for example `31/02/2569`), return `None`;
- do not log, do not raise, do not fall back to "today": the parser has no clock and must stay pure;
- one sentence of docstring stating the conversion, the `None` fallback and the "never raises" rule.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_identity.py -q` — sixteen green.

## Refactor / keep it DRY

The date pattern and the 543 offset live only here; no second copy in the card builder and no "current year" constant elsewhere. Do not add a general date-parsing helper for stage 02's manifest timestamps — those are ISO already. Re-run the suite after the refactor.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_search_parser_identity.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Manual check: print `updated_on` for all 16 cards of page 1; every value is a `date` in 2026 and the first is `2026-10-29`.

## Out of scope (do not do)

- Timezones, "now", relative ages, or sorting by update date — stage 02/03.
- Deriving availability from the update date, or treating a recent date as "available".
- Raising `ParseError` for a bad date (contrast 01-07 and 01-11, where the field is load-bearing).
- Accepting a US-style month-first date, a two-digit year, or a Thai-script month name.

## Definition of done

- [ ] All four cases exist, named as written, and pass; the red run is quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `search_parser.py`, `test_search_parser_identity.py`, this file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
01-14: convert the buddhist-era date to iso

- points: 0.5
- red: test_buddhist_date_is_converted_to_iso failed (ImportError: parse_updated_on)
- add parse_updated_on: day-first date, year minus 543, ISO date
- malformed and missing dates return None without raising
- suite: 81 passed
```

Files in the commit: `search_parser.py`, `test_search_parser_identity.py`, this task file. Nothing else.

## Traps and review notes

- Do not use `datetime.now()` or any clock: the parser is pure and 01-17's snapshot must be byte-identical across runs.
- Do not accept `%d/%m/%Y` blindly and then subtract 543 from whatever year appears: a `01/01/2500` must become 1957, and a year already in the common era would then be mangled — the site's contract is that every printed year is Buddhist-era.
- Do not return `datetime` for a date-only field; the JSON must read `2026-10-29`.
- Do not swallow every exception silently: only the two documented "unreadable" conditions return `None`.
- Reviewers: grep for `now`, `today`, `datetime`, and for a 543 literal outside this function.

## Plan samples are untrusted

`plan/stage-01-search-parser.md` Task 1.4 gives `29/10/2569` and `01/01/2500` as the two cases and says a malformed date yields `None`; there is no code to copy. Verify the installed Python date API you use and record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
