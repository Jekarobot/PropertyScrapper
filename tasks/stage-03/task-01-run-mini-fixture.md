---
id: 03-01
stage: 03
title: Hand-built run-mini raw fixture
points: 1
owner: pipeline
status: todo
depends_on: [01-02]
files:
  - pipeline/tests/fixtures/run-mini/README.md
  - pipeline/tests/fixtures/run-mini/manifest.json
  - pipeline/tests/fixtures/run-mini/pages/page-0001.json
  - pipeline/tests/fixtures/run-mini/pages/page-0002.json
  - pipeline/tests/fixtures/run-mini/pages/page-0003.json
  - pipeline/tests/test_run_mini_fixture.py
---

# 03-01 · Hand-built run-mini raw fixture

**Points:** 1 · **Stage:** 03 — Dataset builder · **Owner lane:** pipeline · **Depends on:** 01-02 · **Unblocks:** 03-05

## Goal

Commit one hand-written raw run directory (`run-mini`) that every dataset-builder test reads, so filtering, dedupe, grouping and statistics are provable without a crawler or the network.

## Why this task exists

Stage 03 turns raw page JSON into the published dataset, and its tests need input small enough to check by hand yet containing every shape that breaks a naive builder: a duplicate unit id at another price, one building in two cases, a name with a comma, unavailable cards reusing an available building's name. The real fixtures are HTML; reading them would drag the network into builder tests.

## Current state (preconditions)

- 01-02 is done: `models.py` defines `UnitCard` and `PageResult` per `plan/02-architecture.md` §3.2; `pipeline/tests/conftest.py` exists (00-05); suite green.
- `pipeline/tests/fixtures/run-mini/` does **not** exist.

## Read before you start

- `CLAUDE.md` §3 (TDD), §6 (plan samples untrusted), §7 (no code in tasks), §10 (no live crawl).
- `plan/02-architecture.md` §3.2 — raw page and manifest fields; §3.3 — the `search` block the manifest `spec` mirrors.
- `plan/01-discovery.md` §3 — `totalPages` / `totalUnits` are 1-based pager values; §2 quirk 2 — location lines split from the right.
- `plan/stage-03-dataset-builder.md` — fixture table and Traps; `models.py` — field names and types.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `pipeline/tests/fixtures/run-mini/pages/page-0001.json` | 4 available cards |
| create | `pipeline/tests/fixtures/run-mini/pages/page-0002.json` | 2 available cards, one duplicate id |
| create | `pipeline/tests/fixtures/run-mini/pages/page-0003.json` | 1 check + 2 not-available cards |
| create | `pipeline/tests/fixtures/run-mini/manifest.json` | run metadata with the FULL spec |
| create | `pipeline/tests/fixtures/run-mini/README.md` | provenance: hand-written, not crawled |
| create | `pipeline/tests/test_run_mini_fixture.py` | the seven cases below |

Nothing else may change: no production module, no `conftest.py` helper, no HTML-fixture edits.

## Test first (red)

1. `test_run_mini_fixture.py::test_three_pages_and_a_manifest` — the directory holds exactly `README.md`, `manifest.json` and `pages/page-0001…0003.json`; each page loads into `PageResult` without error.
2. `test_run_mini_card_counts` — 4 / 2 / 3 cards (nine total); ids per page `[CX-1, CX-2, CX-3, CX-4]`, `[CX-5, CX-1]`, `[CX-6, CX-7, CX-8]`; `position` restarts at 1 on every page.
3. `test_run_mini_page_header_fields` — `schemaVersion` 1, `totalPages` 3, `totalUnits` 9 on all pages; `url` begins `https://connex.in.th/search_en?` with `page_no` equal to `pageNo`; `fetchedAt` is `2026-11-01T21:40:12Z` / `…31Z` / `…50Z`.
4. `test_run_mini_state_counts_match_the_cards` — `{available: 4, check_availability: 0, not_available: 0}`, `{2, 0, 0}`, `{0, 1, 2}`, each equal to the histogram recomputed from that page's cards.
5. `test_run_mini_six_available_cards_five_unique_ids` — six `available` cards (CX-1…CX-5 plus CX-1 again) but five distinct ids; CX-6 the only `check_availability`; CX-7, CX-8 `not_available`.
6. `test_run_mini_manifest_is_a_full_run` — `runId` `run-mini`, `pagesFetched` 3, `requestCount` 3, `errorCount` 0, `stopReason` `last_page`, `partial` `false`, `startedAt` `2026-11-01T21:40:00Z`, `finishedAt` `2026-11-01T21:42:00Z`, `notes` `hand-written, not crawled`, and `spec` = FULL: `project_code` ``, `listingtype` 2, `propertytype` 1, `bedroom` null, `minsize` 45, `maxsize` 150, `minprice` 25000, `maxprice` 50000, `special` null, `sort` ``.
7. `test_run_mini_building_spellings_and_urls` — verbatim `buildingName` values among available cards are exactly `A Tower`, `a tower`, `B Place`, `Condo, A, B`; card `url`s are absolute `https://connex.in.th/en/detail/…` ending in the lowercase id; no available card has null `priceThb`, `sizeSqm` or `bedrooms`.

Expected red: `FileNotFoundError` for the directory (or `AssertionError: run-mini fixture directory does not exist`). A `PageResult` validation error means wrong key names in the fixture, not a model bug.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_run_mini_fixture.py -q`

## Implement (green)

Write the JSON by hand with exactly the camelCase keys of `plan/02-architecture.md` §3.2 (page header, `stateCounts`, and `cards[]` with `id`, `url`, `state`, `priceThb`, `bedrooms`, `bathrooms`, `floor`, `sizeSqm`, `buildingName`, `district`, `province`, `updatedOn`, `imageUrl`, `position`). No HTML, no comments, no extra keys.

| Page.pos | Id | State | Building name (verbatim) | ฿ | sqm | beds | district / province |
|---|---|---|---|---|---|---|---|
| 1.1 | CX-1 | available | `A Tower` | 45000 | 60 | 1 | Sa Thon / Bangkok |
| 1.2 | CX-2 | available | `A Tower` | 50000 | 65 | 1 | Sa Thon / Bangkok |
| 1.3 | CX-3 | available | `B Place` | 32000 | 55 | 2 | Khlong Toei / Bangkok |
| 1.4 | CX-4 | available | `a tower` | 48000 | 58 | 1 | Sa Thon / Bangkok |
| 2.1 | CX-5 | available | `Condo, A, B` | 40000 | 70 | 2 | Khlong Toei / Bangkok |
| 2.2 | CX-1 | available | `A Tower` | 44000 | 60 | 1 | Sa Thon / Bangkok |
| 3.1 | CX-6 | check_availability | `A Tower` | 30000 | 45 | 1 | Sa Thon / Bangkok |
| 3.2 | CX-7 | not_available | `B Place` | 28000 | 40 | 2 | Khlong Toei / Bangkok |
| 3.3 | CX-8 | not_available | `A Tower` | 26000 | 38 | 1 | Sa Thon / Bangkok |

Page 3 reuses `A Tower` / `B Place` on purpose: a leaked excluded card pushes `unitCount` past 3 and `priceMin` below 45000. Cards also carry `bathrooms`, `floor` (floors 12, 20, 8, 15, 5, 12, 3, 2, 4), ISO `updatedOn` and an `imageUrl`, and the README says: hand-written on 2026-11-01, never crawled.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_run_mini_fixture.py -q` — seven green.

## Refactor / keep it DRY

Only test-side duplication exists: one page-loading helper in `test_run_mini_fixture.py`, each expected number once.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_run_mini_fixture.py -q
pytest pipeline -q
ruff check pipeline
```

Manual check: `python -m json.tool pipeline/tests/fixtures/run-mini/pages/page-0001.json` prints formatted JSON.

## Out of scope (do not do)

- No crawler, no `httpx`, no live request, no re-capture of the DEV pages — the fixture is hand-made.
- No reader for the run directory (`load_run` arrives in 03-05); no dataset, grouping or statistics.
- No detail pages, coordinates, `lat`/`lng` or `coordSource`; no expected outputs baked into the files.

## Definition of done

- [ ] All seven cases exist, named as written, and pass; red run quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only the six files above.
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
03-01: add hand-written run-mini raw fixture

- points: 1
- red: test_three_pages_and_a_manifest failed (FileNotFoundError)
- add 3 raw pages (9 cards, 4 spellings, 1 duplicate id), a FULL-spec manifest and README
- add 7 fixture tests
- suite: N passed
```

Files in the commit: the six files above plus this task file (status + Result). Nothing else.

## Traps and review notes

- Do not give page 3 new building names: reusing `A Tower` is what makes an exclusion leak visible.
- The manifest `spec` must be FULL (45/150 sqm, no project code): the dataset `search` block mirrors it verbatim, so a wrong manifest silently rewrites 03-11's assertions.
- `totalUnits` is the site total (9), not this page's cards; and `stateCounts` must match the cards, since a contradiction is indistinguishable from a parser bug. Confirm page-2 CX-1 is priced 44000.

## Plan samples are untrusted

The JSON block in `plan/02-architecture.md` §3.2 is illustrative and shows another unit's card. Take field **names** from it, values from the table above, and validate every page against `PageResult` from 01-02. If the model disagrees, the model wins; record it under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
