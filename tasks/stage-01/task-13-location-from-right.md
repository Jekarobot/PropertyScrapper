---
id: 01-13
stage: 01
title: Building/district/province, parsed from the right
points: 1
owner: pipeline
status: todo
depends_on: [01-12]
files:
  - pipeline/src/connex_watch/search_parser.py
  - pipeline/tests/test_search_parser_identity.py
---

# 01-13 · Building/district/province, parsed from the right

**Points:** 1 · **Stage:** 01 — Search-page parser · **Owner lane:** pipeline · **Depends on:** 01-12 · **Unblocks:** none (03-08 groups on the name)

## Goal

Split the card's location line into building name, district and province by reading from the right, so a comma inside a building name cannot shift the other two.

## Why this task exists

The location line repeats the building name and then lists district and province, separated by the same ` , ` token, and building names really do contain commas. Splitting from the left therefore corrupts the district, which is a user-facing filter and a grouping key in stage 03. Reading from the right is stable because district and province are single segments while the building name is the variable-length prefix. Province must not be dropped either: a two-part line exists in the wild and means "no district", not "province unknown".

## Current state (preconditions)

- 01-05 is done: handles are scoped, and 01-12 is done: `parse_unit_id`, `parse_unit_url`, `parse_image_url` exist with six green cases.
- `UnitCard.building_name` is a non-empty string, `district` is nullable and `province` is a non-empty string (01-02); nothing fills them yet.
- `pytest pipeline -q` is green; no location parser exists.
- Fixture facts: card #1's line is `The Bangkok Sathorn , Sa Thon , Bangkok`; boundary card #4 is `Reflection Jomtien Beach Pattaya , Sattahip , Chon Buri`; boundary card #8 is `DIAMOND TOWER CONDOMINIUM , Bang Rak , Bangkok`.

## Read before you start

- `CLAUDE.md` §3, §5, §6.
- `plan/01-discovery.md` §2 quirk 2 — the location line's shape, the "parse from the right" rule, `building = " , ".join(parts[:-2])`, and the two-part case that yields `district = None`.
- `plan/01-discovery.md` §9 — the unit URL slug must not be used as a building-name source.
- `plan/02-architecture.md` §3.2–§3.3 — `buildingName`, `district` (nullable), `province`; stage 03 normalises names, not this parser.
- `plan/90-test-matrix.md` row P-06 — the requirement this task satisfies.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/search_parser.py` | `parse_location(card_node)` |
| modify | `pipeline/tests/test_search_parser_identity.py` | the six cases below |

Nothing else may change: no name normalisation, no slug, no date, no card assembly.

## Test first (red)

1. `test_location_of_the_first_card` — `search_dev_p1` position 1 → `("The Bangkok Sathorn", "Sa Thon", "Bangkok")`, returned as a three-tuple in that order.
2. `test_multi_word_province_is_kept_whole` — boundary position 4 → `("Reflection Jomtien Beach Pattaya", "Sattahip", "Chon Buri")`; a left-to-right split leaves province as `Chon` and district as `Sattahip` or worse.
3. `test_two_part_line_has_no_district` — replace the first card's line with `HQ Thonglor , Bangkok` (string edit inside the test) → `("HQ Thonglor", None, "Bangkok")`; district is `None`, never `""` and never the province.
4. `test_comma_inside_the_building_name_stays_in_the_name` — replace the line with `A, B Condo , Khlong Toei , Bangkok` → `("A, B Condo", "Khlong Toei", "Bangkok")`; this is matrix row P-06 and the reason the split is from the right.
5. `test_building_name_is_not_title_cased_or_normalised` — boundary position 8 → `DIAMOND TOWER CONDOMINIUM` exactly as printed; normalisation is stage 03's job and the raw layer must show what the site says.
6. `test_extra_whitespace_around_parts_is_trimmed` — replace the line with a padded variant (extra spaces either side of the separators, trailing spaces) → the same three clean values as case 1.

Expected red symptom: `ImportError: cannot import name 'parse_location' from 'connex_watch.search_parser'` for all six cases.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_identity.py -q` — six red, six green.

## Implement (green)

Add `parse_location(card_node: CardNode) -> tuple[str, str | None, str]` to `search_parser.py`:

- read the card's location element text (the `detail_3` line) and trim it;
- split on the exact separator ` , ` (space comma space), trim each part, then read from the right: the last part is the province, the part before it is the district, and the remaining parts joined with ` , ` are the building name;
- three or more parts → all three values; exactly two parts → building and province, district `None`; one part or an empty line → `ParseError("location", position)`, because a card without a province cannot be placed on the map or grouped;
- a building name that itself contains ` , ` is rendered into the same form the site used, so the round-trip in stage 03 keeps working;
- no title-casing, no unicode folding, no slug, no stripping of `Condo`/`The`;
- one sentence of docstring stating the split direction, the nullable district and the error.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_identity.py -q` — twelve green.

## Refactor / keep it DRY

The separator and the right-to-left rule exist once, here. Stage 03's `parse_location`-style splitting (03-04) works on already-split parts and must not re-derive them from HTML; do not add a second splitter now. Re-run the suite after the refactor.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_search_parser_identity.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Manual check: print the three values for all 16 cards of page 1 — all are `The Bangkok Sathorn` / `Sa Thon` / `Bangkok`; then print boundary card 4 and see `Chon Buri` intact.

## Out of scope (do not do)

- Name normalisation, case folding, punctuation or slug generation — stage 03 (03-02, 03-03); district voting, building grouping, or project-code merging — 03-08, 03-10, 04-12.
- Splitting on a bare comma, or from the left "because it looks simpler".
- Deriving the building name from the URL slug, the `h2`, or the image `alt` text.

## Definition of done

- [ ] All six cases exist, named as written, and pass; the red run is quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `search_parser.py`, `test_search_parser_identity.py`, this file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
01-13: parse the location line from the right

- points: 1
- red: test_comma_inside_the_building_name_stays_in_the_name failed (ImportError: parse_location)
- add parse_location: province and district from the right, building as the remaining prefix
- two-part lines yield district None; no title-casing
- suite: 77 passed
```

Files in the commit: `search_parser.py`, `test_search_parser_identity.py`, this task file. Nothing else.

## Traps and review notes

- Do not split on `,` alone: the building name may contain an unpadded comma, and the trimmed parts then come out wrong in a way the page-1 fixture cannot reveal. Do not return an empty string for a missing district — an empty string sneaks through filters as a district named "nothing".
- Do not take the building name from `h2 .title_h2` "because it is cleaner": the raw layer records the location line, and stage 03 compares the two.
- Do not lowercase, strip `The`, or collapse double spaces; the golden snapshot in 01-17 must show the site's strings. Reviewers: grep for `title()`, `lower()` and `replace(",", …)` in this function.

## Plan samples are untrusted

`plan/stage-01-search-parser.md` Task 1.4 gives the expected building/district/province values and the "parse from the right" rule but no code; the discovery doc's quirk 2 is the contract. Verify the separator behaviour against the frozen fixture, and record any departure (for example if you keep an unpadded comma inside a part) under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
