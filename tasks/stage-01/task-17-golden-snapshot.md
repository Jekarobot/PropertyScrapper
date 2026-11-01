---
id: 01-17
stage: 01
title: Golden JSON snapshot of the DEV page
points: 1
owner: pipeline
status: todo
depends_on: [01-11, 01-14, 01-16]
files:
  - pipeline/src/connex_watch/search_parser.py
  - pipeline/tests/test_search_parser_golden.py
  - pipeline/tests/snapshots/search_dev_p1.json
---

# 01-17 · Golden JSON snapshot of the DEV page

**Points:** 1 · **Stage:** 01 — Search-page parser · **Owner lane:** pipeline · **Depends on:** 01-11, 01-14, 01-16 · **Unblocks:** none (stage 02 and 03 consume the parser)

## Goal

Assemble the whole pipeline once — page → 16 cards → pager → `PageResult` — and freeze that output as a committed golden snapshot of `search_dev_p1`.

## Why this task exists

Every field parser exists after 01-16, but nothing has yet built a `UnitCard`: the model requires all fourteen fields, so the card builder could only appear once the last parser did — here. The snapshot is the acceptance evidence of the stage and the regression net for stages 02–04: any markup change, any reordered selector, any accidental normalisation shows up as a readable diff in one committed JSON file, offline, in milliseconds. It also proves the page-level order of decisions: stub first, then pager, then cards.

## Current state (preconditions)

- The lane is done through 01-16: `find_card_nodes`, `parse_state`, `parse_price_thb`, `parse_bedrooms`, `parse_bathrooms`, `parse_floor`, `parse_size_sqm`, `parse_unit_id`, `parse_unit_url`, `parse_image_url`, `parse_location`, `parse_updated_on`, `parse_pager`, `derive_total_units`, `detect_error_stub`, `state_histogram` all exist and are green.
- `PageResult` and `UnitCard` exist (01-02) but nothing constructs a `PageResult` yet; `test_search_parser_golden.py` and `tests/snapshots/` do not exist.
- Fixture facts: `search_dev_p1` has 16 cards, 4 available / 5 check / 7 not available, pager `1 Of 2`; card #1 is `CX-84121`, ฿45,000, 65 sqm, 1 bed, 1 bath, floor 27, `The Bangkok Sathorn , Sa Thon , Bangkok`, updated `29/10/2569`.

## Read before you start

- `CLAUDE.md` §3 (determinism, no clock), §5, §6, §7.
- `plan/02-architecture.md` §3.2 — the raw page JSON this snapshot mirrors, key for key; §6 rules P2 and P3 — stub is end-of-data, zero cards in range is an error (owned by stage 02).
- `plan/stage-01-search-parser.md` Task 1.6 — the snapshot rules, the explicit update command and the exact failure wording.
- `plan/90-test-matrix.md` row P-05 — the histogram the snapshot must agree with.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/search_parser.py` | `parse_card`, `parse_search_page`, the snapshot-comparison helper |
| create | `pipeline/tests/test_search_parser_golden.py` | the six cases below |
| create | `pipeline/tests/snapshots/search_dev_p1.json` | the generated golden file, committed once |

Nothing else may change: no crawler, no dataset, no fixture edit, no `models.py` field.

## Test first (red)

1. `test_golden_snapshot_of_the_dev_page` — `parse_search_page(search_dev_p1).cards` has 16 entries and their `model_dump(mode="json", by_alias=True)` list equals the committed snapshot exactly.
2. `test_snapshot_first_card_matches_the_fixture` — entry 1 reads `id` `CX-84121`, `state` `available`, `priceThb` 45000, `sizeSqm` 65, `bedrooms` 1, `bathrooms` 1, `floor` 27, `buildingName` `The Bangkok Sathorn`, `district` `Sa Thon`, `province` `Bangkok`, `updatedOn` `2026-10-29`, `imageUrl` `…/CX-84121/webp/mini_w500/01_h20zs.webp`, `position` 1.
3. `test_snapshot_page_metadata` — `page_no` 1, `total_pages` 2, `total_units` 32, `is_error_stub` `False`.
4. `test_parsed_page_histogram_is_four_five_seven` — `state_histogram` over the parsed cards gives 4 / 5 / 7, proving the page-level assembly did not drop or duplicate cards.
5. `test_snapshot_mismatch_message_names_the_next_step` — the comparison helper, given a deliberately altered dump, raises with the exact sentence `parser output changed; if the site changed, re-capture the fixture and review the diff; if not, fix the parser`.
6. `test_error_stub_page_parses_to_no_cards` — `parse_search_page(search_error_page)` → `cards == []`, `is_error_stub is True`, `page_no` and `total_pages` `None`, and no exception — the integration half of 01-08.

Expected red symptom: `ImportError: cannot import name 'parse_search_page' from 'connex_watch.search_parser'` for cases 1–4 and 6; the missing snapshot file is reported as `FileNotFoundError` in case 1.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_golden.py -q` — six red.

## Implement (green)

- `parse_card(card_node: CardNode) -> UnitCard` — call the eleven field parsers exactly once and pass every field explicitly, including the nullable ones; no defaults beyond what 01-02 requires, no post-processing, no reordering.
- `parse_search_page(html: str) -> PageResult` — in this order: classify the stub first (`detect_error_stub` → `PageResult(cards=[], is_error_stub=True)` with `page_no`/`total_pages`/`total_units` `None`), then parse the pager, then build cards from `find_card_nodes` and set `total_units` from `derive_total_units(page_no, total_pages, len(cards))`. Pure: no I/O, no clock, no globals.
- the snapshot comparison helper loads the committed JSON once, compares the full list, and raises with the sentence from case 5, followed by the first differing card index and its two values.
- snapshot writing happens **only** when `UPDATE_SNAPSHOTS=1` is set in the environment; no `--write-snapshot` flag, no rewriting on failure, no `pytest` hook. Generate the file by running the test once with that variable, then commit it and run the suite again without it.
- one sentence of docstring per public item.

Run: `. .venv/bin/activate && UPDATE_SNAPSHOTS=1 pytest pipeline/tests/test_search_parser_golden.py -q` once to create the snapshot, then `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_golden.py -q` — six green.

## Refactor / keep it DRY

`parse_card` is the only place a `UnitCard` is constructed and the only caller of the field parsers in sequence; no field may be parsed twice (a second call is the beginning of a divergence). The page builder owns the stub-then-pager order once. Re-run the suite after the refactor.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_search_parser_golden.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Manual check: run the suite twice and `git status --short` — the snapshot must be unchanged after the second run; then `git diff` it and read the sixteen cards.

## Out of scope (do not do)

- Fetching, crawling, pagination walking, checkpoints or a run directory — stage 02; filtering to `available`, grouping by building or writing a dataset — stage 03; detail-page coordinates or project codes — stage 04.
- Regenerating the snapshot automatically on failure, or a `--snapshot-update` flag next to the env var.
- Asserting an exact byte size or a hash of the fixture.

## Definition of done

- [ ] All six cases exist, named as written, and pass; the red run is quoted in the commit body.
- [ ] The snapshot file is committed, re-running the suite leaves it unchanged, `pytest pipeline -q` is green (≥ 25 tests, offline, under two seconds) and `ruff check pipeline` is clean.
- [ ] `git status --short` lists only `search_parser.py`, `test_search_parser_golden.py`, the snapshot, this file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash, test counts and the snapshot's unit count.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
01-17: add the golden snapshot of the dev search page

- points: 1
- red: test_golden_snapshot_of_the_dev_page failed (ImportError: parse_search_page)
- add parse_card and parse_search_page: stub first, then pager, then 16 cards
- commit the 16-card snapshot, writable only via UPDATE_SNAPSHOTS=1
- suite: 96 passed
```

Files in the commit: `search_parser.py`, `test_search_parser_golden.py`,
`pipeline/tests/snapshots/search_dev_p1.json`, this task file. Nothing else.

## Traps and review notes

- If the snapshot is regenerated automatically on failure it proves nothing; the update path must require a deliberate environment variable.
- Do not normalise anything on the way into the snapshot (no title-casing, no rounding, no `None` → `""`): the diff is the alarm.
- Do not sort the cards; `position` order is the site's order and the array order must match it.
- Do not let the stub path fall through to the pager: the stub has no pager and would raise `ParseError("pagination")` instead of reporting end-of-data.
- Reviewers: grep the golden test for a `pytest` hook, `write_text` outside the env-var branch, and any call to the field parsers that is not inside `parse_card`.

## Plan samples are untrusted

`plan/stage-01-search-parser.md` Task 1.6 shows the comparison as `[c.model_dump(mode="json") for c in cards]`; without `by_alias=True` that produces snake_case keys which do not match the architecture's raw-page JSON, so this task requires the alias form. Verify the pydantic dump options against the installed version and record the deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
