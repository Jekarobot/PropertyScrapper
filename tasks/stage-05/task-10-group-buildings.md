---
id: 05-10
stage: 05
title: Group filtered units, recompute ranges, labels
points: 1
owner: web
status: todo
depends_on: [05-03]
files:
  - web/src/lib/group.ts
  - web/src/lib/group.test.ts
---

# 05-10 · Group filtered units, recompute ranges, labels

**Points:** 1 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 05-03 · **Unblocks:** 05-16, 06-06

## Goal

Add `groupByBuilding(ds, units)` that returns the dataset's buildings in dataset order with unit counts and price/size ranges recomputed from the filtered units, plus the `priceRange` and `countLabel` display helpers.

## Why this task exists

The rows the user reads are buildings, but every number on them must describe what passed the filters, not the whole building. The published dataset ships precomputed `unitCount`/`priceMin`/`priceMax`/`sizeMin`/`sizeMax`, and trusting them after filtering is the most likely correctness bug in the UI: a building with three units would still say `3 units · 45–50k` while showing one 45,000 unit. This module makes the recomputation the only way to get a row, and gives the row markup its two labels so they are not formatted independently in two components (W-12).

## Current state (preconditions)

- 05-03 is done: the fixture has three buildings, six units, and per-building summaries that match the full unit lists; `web/src/types.ts` exports `Dataset`, `Building`, `Unit`.
- `web/src/lib/` contains `filter.ts` (with its test); there is no `group.ts`.
- The fixture's `p00348` has three units; `CX-84121` is 45,000 THB / 65 sqm.

## Read before you start

- `CLAUDE.md` §3 (exact fixture numbers), §5 (one implementation, no `any`), §6, §7.
- `plan/02-architecture.md` §3.3 (per-building summary fields and the sorting rules), §5.1 (row shows name, district, count and price range), §5.2 F5.
- `plan/stage-05-web-shell.md` Task 5.3 — `group.ts`'s three exports and the "recomputed ranges, not dataset values" case.
- `plan/90-test-matrix.md` row W-12 — "rows show recomputed ranges, not dataset ranges"; row W-05.
- `web/src/test/fixtures/dataset.sample.json` — read `p00348`'s units and its precomputed summary before writing the tests.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `web/src/lib/group.ts` | `groupByBuilding`, `priceRange`, `countLabel` |
| create | `web/src/lib/group.test.ts` | the six cases below |

Nothing else may change: no component, no hook, no `filter.ts` edit, no fixture edit.

## Test first (red)

Import the fixture, look up units by id and pass subsets to `groupByBuilding`. Build `Building` variants for `priceRange` by spreading the fixture building and overriding only the two price fields.

1. `test_groupByBuilding_recomputes_ranges_from_the_filtered_units` — pass only `CX-84121` (45,000 / 65 sqm): the result is one building (`p00348`) with `unitCount` 1, `priceMin`/`priceMax` 45000 and `sizeMin`/`sizeMax` 65 — and these differ from the fixture's precomputed 3 / 45000 / 50000 / 58 / 65, which is the point of the test.
2. `test_groupByBuilding_returns_empty_array_for_no_units` — `[]` → `[]` (not three empty buildings).
3. `test_groupByBuilding_preserves_dataset_order_and_drops_empty_buildings` — pass units from `premier-thonglor` and `p00348` interleaved; the output ids are `p00348`, `premier-thonglor` in that order, `no-location` is absent, and each building's `units` keep the order they had in the passed array.
4. `test_groupByBuilding_does_not_mutate_the_dataset` — after the call the fixture building still has `unitCount` 3 and three units, and the input unit array still has its original length; the result's building objects are new objects.
5. `test_priceRange_labels_span_and_single_values` — table: (45000, 50000) → `45–50k`; (45000, 45000) → `45k`; (1200000, 1200000) → `1.2M`; (1000000, 1200000) → `1–1.2M`; (999, 999) → `999`; (1000, 1000) → `1k`; (45500, 45500) → `45.5k`; (999, 1000) → `999–1k`; (45000, 1200000) → `45k–1.2M`. The separator is an en dash (U+2013), never a hyphen.
6. `test_countLabel_is_singular_only_for_one` — 1 → `1 unit`; 3 → `3 units`; 0 → `0 units`; 2 → `2 units`.

Expected red symptom: `Cannot find module './group'`. If a case fails with a type error after the module exists, fix the case before changing the module.

Run: `pnpm -C web test group` — six red for the missing module.

## Implement (green)

- `groupByBuilding(ds, units)`: bucket the passed units by `building.id`, then walk `ds.buildings` in order and emit a shallow copy for every building that has at least one unit, skipping the rest. The copy keeps `id`, `name`, `district`, `province`, `lat`, `lng`, `coordSource` and replaces `units` plus the five summary fields: `unitCount` = passing units count, `priceMin`/`priceMax` = min/max `priceThb`, `sizeMin`/`sizeMax` = min/max `sizeSqm`. Never read the dataset's precomputed summary fields. Never mutate the dataset or the units; return new objects and a new array.
- `priceRange(b)`: format each bound as a magnitude plus a unit — below 1 000 the integer with no suffix; 1 000…999 999 → value/1000 rounded half-up to one decimal with a trailing `.0` removed and suffix `k`; 1 000 000 and above → the same with suffix `M`. Equal bounds collapse to one value (`45k`, `1.2M`, `999`). For a span, when both bounds share the unit, print the magnitudes once and one trailing suffix (`45–50k`, `1–1.2M`); when the units differ, print each bound fully (`999–1k`, `45k–1.2M`). Join with an en dash (U+2013) and no spaces. No `Intl`, no `toLocaleString`, no currency symbol (that is 05-12's `thb`).
- `countLabel(n)`: `n` followed by `unit` when `n === 1`, otherwise `units`; no thousands separators, no pluralisation library.
- One-sentence docstrings on all three; no `any`; no React import; no sorting (dataset order is already the contract, §3.3).

Run: `pnpm -C web test group` — six green.

## Refactor / keep it DRY

The five summary fields are computed in one place; `priceRange` is the only short-price formatter and is not re-implemented in the component that renders a row (06-06 imports it). `countLabel` is the only pluraliser; the header count (05-19) and the no-location bucket (07-10) must call it rather than concatenating strings. Do not add a group-by-district or sorting mode — neither has a caller.

## Verification (how to test it)

```text
pnpm -C web test group
pnpm -C web test --run
pnpm -C web typecheck
pnpm -C web build
make check
```

Expected: 6 passed for the target, 53 for the lane, typecheck 0, build clean, `make check` green. Manual: `grep -n 'unitCount\|priceMin\|priceMax\|sizeMin\|sizeMax' web/src/lib/group.ts` shows assignments, never reads of `b.unitCount`-style fields.

## Out of scope (do not do)

- No sorting, no filtering, no viewport logic, no selection, no component or hook.
- No currency or area formatting (`thb`, `sqm`, `beds` are 05-12); no `Intl` usage anywhere in this file.
- No caching or memoisation; no extra exports beyond the three names.

## Definition of done

- [ ] All six cases exist in `web/src/lib/group.test.ts`, named as written, and pass.
- [ ] Red run observed and quoted; the range case proves recomputation by differing from the fixture summary.
- [ ] `pnpm -C web typecheck` clean; lane suite and `make check` green; no `any`.
- [ ] Only the two listed files are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled; exactly one commit on the trunk (`main`) with the message below.

## Commit

```text
05-10: group filtered units and recompute building ranges

- points: 1
- red: group.test.ts::test_groupByBuilding_recomputes_ranges_from_the_filtered_units failed (Cannot find module './group')
- add groupByBuilding with recomputed counts and price/size ranges
- add priceRange and countLabel display helpers
- suite: 53 passed
```

Files in the commit: `web/src/lib/group.ts`, `web/src/lib/group.test.ts`, and this task file.

## Traps and review notes

- Reading the dataset's `unitCount`/`priceMin` instead of recomputing makes case 1 pass only if the fixture summary happens to match; the test passes a single unit precisely to make that impossible.
- `Math.min` of an empty array is `Infinity`; filter empty buildings out before computing, or the row shows `Infinity`.
- Use an en dash U+2013 in `priceRange`: a hyphen passes a visual check and fails the exact string assertion, and the row would not match the design.
- Do not format prices with `toLocaleString`/`Intl`: output must be identical on every machine and in tests.
- Reviewers: `grep -n 'b\.unitCount\|building\.priceMin' web/src/lib/group.ts` empty; `grep -n '45–50k' web/src/lib/group.test.ts` present.

## Plan samples are untrusted

The plan lists the three exports and sample labels but no bodies; its numbers are illustrative. Verify the short-price rule and the en dash against the task (the task is the contract) and do not paste code from the plan (`CLAUDE.md` §6). If a plan sample disagrees on the label format, record the deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Deviations | |
