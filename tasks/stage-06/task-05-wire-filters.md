---
id: 06-05
stage: 06
title: Wire the filter bar into App state
points: 1
owner: web
status: todo
depends_on: [06-04, 05-17]
files:
  - web/src/App.tsx
  - web/src/App.test.tsx
---

# 06-05 · Wire the filter bar into App state
**Points:** 1 · **Stage:** 06 — Filters and results list · **Owner lane:** web · **Depends on:** 06-04, 05-17 · **Unblocks:** 06-12

## Goal
Mount `FilterBar` in `App`, connect `filters`/`onChange`/`onReset`/`resultCount` to the state hooks, and prove that filtering changes both the reported count and the buildings shown.

## Why this task exists
Until now the bar is a well-tested orphan and `App` renders a fixed list. This task closes the loop that is the product's core interaction: change a filter, the list and the count change, without a page reload. It is deliberately thin — one
composition point, no new logic, everything derived from `lib/filter.ts` and `lib/group.ts` — because the rules already exist and were tested in stage 05; what can go wrong here is wiring: a count computed from buildings instead of units, a
reset that leaves a ghost selection, or a recomputed range that still shows hidden prices.

## Current state (preconditions)
- 06-04 done: `FilterBar` renders all five controls, Reset and the count; its props interface is final.
- 05-17 done: `App.tsx` composes `useDataset`, `useFilters`, `useViewport`, `useSelection` and derives units/buildings from the sample dataset; it renders a placeholder results list (one item per building, names only).
- 05-07/05-09/05-10 done: `lib/filter.ts` exports `filterUnits` and `filterBuilding`; `lib/group.ts` exports `groupByBuilding`, `countLabel`, `priceRange`; `lib/format.ts` exports `sqm`.
- 05-12/05-13 done: the test fixture `dataset.sample.json` and the `useDataset` loading hook exist.

## Read before you start
- `plan/stage-06-filters.md` Task 6.2 (all six cases) and the last Trap (never display dataset ranges after filtering).
- `plan/02-architecture.md` §5.2 rows F3, F4, F5 (filters before the viewport cut; a building survives if ≥ 1 unit passes) and §5.1 (left panel vs right list).
- `plan/90-test-matrix.md` rows W-05 (filter before viewport — proven in 07-06), W-12 (recomputed ranges).
- `web/src/lib/filter.ts`, `lib/group.ts`, `web/src/App.tsx`, `web/src/App.test.tsx` — read what already exists; the wiring must reuse it.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/App.tsx` | mount the bar, pass derived props, express reset |
| modify | `web/src/App.test.tsx` | the six cases below |

Nothing else may change: no component internals, no `lib/` changes, no new file.

## Test first (red)
Read every expected count and building name from `dataset.sample.json` and write it as a literal; never call `filterUnits` in the test to produce its own expectation. Query the list by its list-item role and visible text, not by test ids —
06-09 replaces the markup and these tests must survive it.

1. `test_unfiltered_state_reports_six_units_and_three_buildings` — the bar shows `6 found` and the list has three items, one per building (6 units / 3 buildings in the fixture).
2. `test_beds_two_keeps_only_two_bedroom_units_and_drops_emptied_buildings` — choose `2`: the count equals the literal number of two-bedroom units in the fixture, and the listed buildings are exactly those with at least one such unit; a
   building with no two-bedroom unit is absent.
3. `test_size_min_sixty_removes_the_45_sqm_unit_and_recomputes_ranges` — type `60` in the minimum field and advance the debounce: the count drops by exactly one versus unfiltered, the building that held the 45 m² unit shows its recomputed unit
   count and its recomputed size range (literals from the remaining units), and it disappears entirely if that was its only unit.
4. `test_district_watthana_keeps_only_premier_thonglor` — choose `Watthana`: exactly one row, named `Premier Thonglor`, count equal to its unit count.
5. `test_building_text_matches_case_insensitively` — type `bangkok` (lowercase): the rows are exactly the buildings whose name contains it case-insensitively — `The Bangkok Sathorn` present, `Premier Thonglor` absent.
6. `test_reset_restores_the_full_list_and_clears_the_selection` — select a row (it gains `aria-current="true"`), then click `Reset`: three rows again, `6 found`, and no row carries `aria-current`.

Expected red: `Unable to find a label with the text of: Size minimum` inside `App`, or a count assertion showing the fixed `6 found` while a filter is set. Run `pnpm -C web test --run App`, quote one line.

## Implement (green)
Replace the disabled placeholder in `App.tsx` with the real bar and derive the list from state. All rules already live in `lib/`:

- Filter per building with `filterBuilding(building, filters)` — it is the only function with the building context F3/F4 need (district, name) — drop `null` results, then recompute with `groupByBuilding` so `unitCount`, `priceMin/Max`,
  `sizeMin/Max` describe the surviving units. If `lib/filter.ts` lacks `filterBuilding`, stop and report; do not reimplement F3/F4 in `App`.
- `resultCount` is the number of surviving **units** (sum of `unitCount`), the same unit the bar's `N found` shows; do not pass the building count.
- `onChange` forwards to `setFilter`; `onReset` calls `reset()` **and** `select(null)`, because `useSelection` only drops a selection when the building leaves the set and a reset re-adds it — the ghost panel is exactly this bug.
- `districts` is the sorted unique district list from the dataset (build once, not per render); `buildingOptions` is the first 50 building names (the bar enforces the ceiling; keep the source list deterministic).
- The placeholder row may show name, `countLabel`, `priceRange` and the max size via `sqm` so case 3 is observable; that markup is temporary and is deleted in 06-09. No new formatting logic.
- No viewport cut in this task (`bbox` stays unused or `null` = whole city).

Run `pnpm -C web test --run App` — six new cases green, earlier `App` cases still green.

## Refactor / keep it DRY
`App` computes, components render: no `if (filters.beds === …)` anywhere in `App` or the bar. If a derived value is computed twice (e.g. districts in `App` and in the bar), keep it in `App` once and pass it down. Memoise the district and
option lists only if a test proves a cost; otherwise keep it simple.

## Verification (how to test it)
```text
pnpm -C web test --run App
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: the six cases green; full web suite green; `tsc --noEmit` clean, no `any`; `make check` exits 0.

## Out of scope (do not do)
- The viewport cut and its ordering proof (W-05, 07-06); map, bubbles, panels (stage 07).
- `ResultsList`/`BuildingRow` components (06-06, 06-09); the counts header (06-10); error/empty panels (06-12, except where the existing placeholder already renders something).
- URL sync, debounce inside `App`, per-keystroke filtering, fuzzy matching, new dependencies.

## Definition of done
- [ ] Six cases exist, named as written, and pass; no earlier `App` test was edited to accommodate them.
- [ ] Red run observed and quoted; the reset case asserts the absent `aria-current`, not just the rows.
- [ ] Web suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the two files above.
- [ ] `status: done`; `## Result` filled; one commit on `main`.

## Commit
```text
06-05: wire filters into the app

- points: 1
- red: test_unfiltered_state_reports_six_units_and_three_buildings failed (size controls not found in App)
- mount FilterBar and derive buildings with filterBuilding plus groupByBuilding
- resultCount is units; reset also clears the selection
- suite: N passed
```

Commit this task alone, with the listed files only, on the trunk (`main`) — one task = one
commit, no branch, no push.

## Traps and review notes
- Passing the building count as `resultCount` makes `12 found` mean buildings and quietly contradicts the header in 06-10; the units assertion exists for this.
- A reset that only calls `reset()` leaves the left panel showing a building that is back in the list — the "ghost panel" bug the plan warns about; clear the selection explicitly.
- Displaying `building.priceMin` from the dataset after filtering shows prices of hidden units (W-12); always use the grouped/recomputed object.
- Recomputing districts/building options inside render on every keystroke is a measurable cost later; keep them derived from the dataset, not from `filters`.
- Reviewer greps `App.tsx` for filter comparisons and for `filterUnits(` on raw dataset units (missing building context).

## Plan samples are untrusted
The plan's composition sketch (`units = filterUnits(...)`, then grouping) ignores the building context F3/F4 need; follow the repo's `filterBuilding` and record the deviation under `## Decisions`. Verify hook signatures against
`web/src/state/*.ts` before trusting the sketch (`CLAUDE.md` §5, §6).

## Decisions
## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
