---
id: 02-08
stage: 02
title: Count duplicate unit ids, do not drop here
points: 0.5
owner: pipeline
status: todo
depends_on: [02-05]
files:
  - pipeline/src/connex_watch/crawl.py
  - pipeline/tests/test_crawl_stop.py
---

# 02-08 · Count duplicate unit ids, do not drop here

**Points:** 0.5 · **Stage:** 02 — Crawler · **Owner lane:** pipeline · **Depends on:** 02-05 · **Unblocks:** 03-06

## Goal
Record how many cards repeated a unit id already seen in the same run, without removing anything from the raw result.

## Why this task exists
The search ranking is stable seconds apart but changes between runs, and the site paginates a changing list, so one unit can legitimately appear on two pages of one crawl (`plan/01-discovery.md` §6, risk R6). Deduplicating inside the crawler would change the raw layer, hide a pagination regression and make the "duplicate count" in the run summary unexplainable. The raw layer keeps every card and only counts repeats; stage 03 decides first-wins dedupe when it builds the dataset and needs the count to report what it removed.

## Current state (preconditions)
- 02-05 is done: the walk accumulates `cards` in page order; `CrawlResult.duplicate_units` exists with its documented meaning (extras, not distinct ids) defaulting to `0`.
- Nothing computes `duplicate_units` yet; every run reports `0`.
- `test_crawl_stop.py` is green with twenty-one cases, `FakeFetcher` and `page(body, n, total)`.
- Unit ids are available as `UnitCard.id` in the normalised `CX-<digits>` form (01-12).

## Read before you start
- `CLAUDE.md` §3 (one behaviour per test), §5 (DRY: a rule has one home), §6.
- `plan/01-discovery.md` §6 — ranking changes, "deduplicate by unit id anyway", risk R6.
- `plan/02-architecture.md` §3.1 (`UnitCard` identity is `CX-<digits>`) and §6 rule P8.
- `plan/stage-02-crawler.md` Task 2.2 row 10; `plan/90-test-matrix.md` row P-10.

## Files to touch
| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/crawl.py` | the seen-id set and the duplicate counter |
| modify | `pipeline/tests/test_crawl_stop.py` | the three cases below |

Nothing else: no `models.py` change (the field exists), no dedupe in the crawler, no logging.

## Test first (red)
1. `test_crawl_stop.py::test_duplicate_unit_ids_across_pages_are_counted_not_dropped` — page 1 = `search_dev_p1` (`1 Of 5`), page 2 = the **same** body with pager `2 Of 5`, `max_pages=2` → `len(cards) == 32`, `duplicate_units == 16`, and the 16 fixture ids each appear exactly twice; every card is kept.
2. `test_crawl_stop.py::test_duplicate_counting_does_not_change_the_histogram` — same fake and policy → `state_histogram == {available: 8, check_availability: 10, not_available: 14}`; both copies are counted, so the histogram still totals `len(cards) == 32`.
3. `test_crawl_stop.py::test_dev_walk_reports_zero_duplicates` — the DEV three-page fake from 02-05 → `duplicate_units == 0`, `len(cards) == 32`, and `len({card.id for card in cards}) == 32`; the counter is not a constant.

Expected red symptom: `assert 0 == 16` for case 1 and `assert 0 == 32` for case 3, because the field exists but nothing fills it.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_crawl_stop.py -q`.

## Implement (green)
- While cards accumulate, keep the seen unit ids in one set; for each card, increment `duplicate_units` by one when its id is already present, then add the id. The set lives on the crawler instance — never a module global, or a second `crawl()` in the same process starts polluted.
- Nothing is removed, reordered or merged: `cards` keeps every card in page order and the histogram keeps counting every card.
- `duplicate_units` is written into the final `CrawlResult`; it is never recomputed from `pages` afterwards, so a resumed run can add to it (02-11).
- One comment on the field meaning is enough; no helper class, no `Counter`, no per-duplicate logging, no new import.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_crawl_stop.py -q` — twenty-four green.

## Refactor / keep it DRY
The "already seen" rule lives here and only here; stage 03 implements its own first-wins decision in `build_dataset.py` because that rule is about the dataset, not about counting. Do not export the set or a dedupe helper. Re-run the suite.

## Verification (how to test it)
```text
. .venv/bin/activate && pytest pipeline/tests/test_crawl_stop.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 24 passed, the pipeline suite green, `ruff` exits 0. Manual check: none.

## Out of scope (do not do)
- Deduplication, first-wins selection, dropping a card whose id was seen — 03-06.
- Duplicate detection for buildings or by URL; only unit ids are counted.
- A per-page duplicate count, a duplicate list in the result, per-duplicate logging.
- Excluding duplicates from the state histogram. The CLI summary field — 02-13 / 02-14.

## Definition of done
- [ ] All three cases exist, named as written, and pass; the earlier cases still pass.
- [ ] Red run observed and quoted in the commit body.
- [ ] Target tests + `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `crawl.py`, `test_crawl_stop.py` and this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit
```text
02-08: count duplicate unit ids without dropping cards

- points: 0.5
- red: test_duplicate_unit_ids_across_pages_are_counted_not_dropped failed (assert 0 == 16)
- track seen unit ids per run and count the repeats
- keep every card and its state in the raw result
- suite: 49 passed
```

Files in the commit: `pipeline/src/connex_watch/crawl.py`, `pipeline/tests/test_crawl_stop.py`, and this task file — nothing else.

## Traps and review notes
- Do not deduplicate "while you are here": the raw layer keeps duplicates and stage 03 needs the count to explain what it removes.
- Do not count distinct repeated ids (16 repeated ids are 16 extras, not one); the definition lives on `CrawlResult`.
- Do not store the set on the class or the module; a second `crawl()` would report false duplicates.
- Reviewer: case 1 must assert both `len(cards)` and `duplicate_units` — a counter that also filters passes a duplicate-only assertion.

## Plan samples are untrusted
The plan names `CrawlResult.duplicate_units` but not its counting rule; this task pins "extras, not distinct ids" in the tests. Check the field name in the repo's `models.py` before coding and record a deviation under `## Decisions`.

## Decisions
<!-- append during implementation; leave empty at planning time -->

## Result
| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
