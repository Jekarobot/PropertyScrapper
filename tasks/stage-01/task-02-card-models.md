---
id: 01-02
stage: 01
title: "`UnitCard`/`PageResult` models and validation"
points: 1
owner: pipeline
status: todo
depends_on: [01-01]
files:
  - pipeline/src/connex_watch/models.py
  - pipeline/tests/test_models.py
---

# 01-02 · `UnitCard`/`PageResult` models and validation

**Points:** 1 · **Stage:** 01 — Search-page parser · **Owner lane:** pipeline · **Depends on:** 01-01 · **Unblocks:** 01-03, 01-05, 02-04, 03-01, 03-02, 04-01

## Goal

Add the three pydantic models every later layer exchanges: one result card, the page envelope that holds cards, and the search spec with the production defaults.

## Why this task exists

The crawler's checkpoints, the dataset builder and the golden snapshot all exchange the same two shapes: a card and a page of cards. If those shapes are loose — price as text, floor defaulting to 0, state as a free string — every consumer re-validates the same assumptions differently and a "0 THB" unit reaches the map. Fixing the model with validation is what lets the parsers stay dumb: a parser that returns a card has already proven its numbers are plausible. `PageResult` is declared here but first assembled in 01-17.

## Current state (preconditions)

- 01-01 is done: `models.py` has `UnitState` + `from_label`; `test_models.py` has four green cases.
- `pytest pipeline -q` is green and `.venv` exists at the repo root.
- `UnitCard`, `PageResult`, `SearchSpec` do not exist; nothing constructs them yet (parser 01-05, assembly 01-17). Every value below is copied from the frozen fixtures — read the fixture, do not invent.

## Read before you start

- `CLAUDE.md` §2, §3, §4, §6, §7.
- `plan/02-architecture.md` §3.1 — domain objects and "only `available` is ever published"; §3.2 — the raw-page JSON that fixes every key name.
- `plan/02-architecture.md` §3.3 — the published `search` block is snake_case and has no page number.
- `plan/assets/fixtures/README.md` — card #1: `CX-84121`, `฿45,000`, 65 sqm, 1 bed, 1 bath, floor 27, `The Bangkok Sathorn , Sa Thon , Bangkok`, updated `29/10/2569`.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/models.py` | `SearchSpec`, `UnitCard`, `PageResult` |
| modify | `pipeline/tests/test_models.py` | the seven cases below |

Nothing else may change: no exports, no parser, no URL code, no conftest edits.

## Test first (red)

1. `test_unit_card_accepts_a_complete_card` — card #1's real values → every attribute reads back unchanged, `state is UnitState.AVAILABLE`.
2. `test_unit_card_rejects_non_positive_price` — price 0, then −1 → `ValidationError` both times.
3. `test_unit_card_allows_null_bathrooms_floor_and_district` — the three pass as `None` and stay `None`, not 0 or `""`.
4. `test_unit_card_dump_uses_the_raw_page_keys` — `model_dump(mode="json", by_alias=True)` yields the §3.2 keys in the order `id, url, state, priceThb, bedrooms, bathrooms, floor, sizeSqm, buildingName, district, province, updatedOn, imageUrl, position`, with `state` as `available`, `updatedOn` as `2026-10-29`.
5. `test_size_keeps_int_for_whole_numbers_and_float_for_decimals` — 65 dumps as `65`, 55.5 as `55.5`; a float-typed field would serialise every real card as `65.0`.
6. `test_page_result_defaults_describe_a_healthy_page` — `PageResult(cards=[card])` leaves `page_no`/`total_pages`/`total_units` `None`, `is_error_stub` `False`.
7. `test_search_spec_defaults_are_the_full_query` — `SearchSpec()` gives `project_code=""`, `listingtype=2`, `propertytype=1`, `bedroom=None`, `minsize=45`, `maxsize=150`, `minprice=25000`, `maxprice=50000`, `special=None`, `sort=""`; its dump keys stay snake_case.

Expected red symptom: `ImportError: cannot import name 'UnitCard' from 'connex_watch.models'`; a later value mismatch means the model is wrong, not the test.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_models.py -q` — four green, seven red.

## Implement (green)

In `models.py`, leaving `UnitState` untouched:

- `SearchSpec` — frozen, snake_case, no URL methods: `project_code: str = ""`, `listingtype: int | None = 2`, `propertytype: int | None = 1`, `bedroom: int | None = None`, `minsize: int | None = 45`, `maxsize: int | None = 150`, `minprice: int | None = 25000`, `maxprice: int | None = 50000`, `special: int | None = None`, `sort: str = ""`. `None` stores an empty numeric parameter, `""` an empty string parameter.
- `UnitCard` — `id: str`, `url: str`, `state: UnitState`, `price_thb: int` strictly `> 0`, `bedrooms: int >= 0`, `bathrooms: int | None`, `floor: int | None`, `size_sqm: int | float > 0`, `building_name: str` non-empty, `district: str | None`, `province: str` non-empty, `updated_on: date | None`, `image_url: str`, `position: int >= 1`; camelCase aliases per §3.2.
- `PageResult` — `cards: list[UnitCard]` (required, may be empty), `page_no`/`total_pages`/`total_units` as `int | None = None`, `is_error_stub: bool = False`; aliases `cards, pageNo, totalPages, totalUnits, isErrorStub`.
- `bathrooms`, `floor`, `district`, `updated_on` are nullable but still required arguments: the parser decides `None`, the model never defaults it. One-sentence docstring per model and on the price/floor/size bounds; `schemaVersion`, `fetchedAt`, `url` and `stateCounts` are not here — the raw-page writer (02-10) composes them.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_models.py -q` — eleven green.

## Refactor / keep it DRY

The camelCase keys exist once — one alias generator or shared config on the two card/page models, never fourteen hand-written strings — and `SearchSpec` stays snake_case. The `> 0` price bound lives only in the model. Re-run the suite after the refactor.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_models.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Manual check: dump one card with `by_alias=True`; the fourteen keys appear in the order of case 4 and none is missing.

## Out of scope (do not do)

- `from_url`, `to_url`, `.extra`, `.with_()` — 01-03 and 01-04 own them.
- Any HTML parser, selector, `ParseError` or histogram — 01-05 onward.
- A `stateCounts` field — composed by the raw-page writer (02-10), not stored here.
- An `id`/`image_url` regex (01-12 normalises), an `UNKNOWN` state, or an `is_available` helper.

## Definition of done

- [ ] All seven cases exist, named as written, and pass; the red run's `ImportError` line is quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `models.py` and `test_models.py`.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
01-02: add unit card and page result models

- points: 1
- red: test_unit_card_accepts_a_complete_card failed (ImportError: UnitCard)
- add SearchSpec with FULL-query defaults, UnitCard with price/floor/size bounds, PageResult
- camelCase aliases on card and page only; SearchSpec stays snake_case
- suite: 11 passed
```

Files in the commit: `models.py`, `test_models.py`, this task file. Nothing else.

## Traps and review notes

- Do not camel-case `SearchSpec`: stage 03 dumps it straight into the snake_case `search` block. Do not default `floor` or `bathrooms` to 0 — 0 is a real floor on the boundary fixture, `None` means "not stated".
- Do not type `size_sqm` as `float`: every real card would serialise as `65.0` and 01-17's snapshot would diverge from the architecture's JSON.
- Do not validate `id` shape here; 01-12 normalises it and would then fight this model.
- Reviewers: grep `models.py` for parsing helpers or `try/except` — there must be none.

## Plan samples are untrusted

`plan/stage-01-search-parser.md` Task 1.1 names the models but shows no usable code; the field list in `plan/02-architecture.md` §3.2 is the contract. Verify the pydantic v2 behaviour you rely on (aliases, frozen models, `Field(gt=…)`, `model_dump(mode="json", by_alias=True)`) against the installed version and record deviations under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
