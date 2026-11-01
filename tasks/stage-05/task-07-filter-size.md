---
id: 05-07
stage: 05
title: Size bounds filter and filterBuilding null
points: 1
owner: web
status: todo
depends_on: [05-03]
files:
  - web/src/lib/filter.ts
  - web/src/lib/filter.test.ts
---

# 05-07 · Size bounds filter and filterBuilding null

**Points:** 1 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 05-03 · **Unblocks:** 05-08, 05-14

## Goal

Add the filter module: the `Filters` type from the plan and a pure size-bounds predicate that ignores unusable bounds, plus `filterBuilding` returning null when nothing passes.

## Why this task exists

Filtering is the product's core interaction and the one rule every panel depends on: the results list, the map, the header count and the selected-building panel all show a filtered subset. `plan/02-architecture.md` §5.2 F1 fixes the size semantics — inclusive bounds, empty means no bound, invalid input ignored rather than turned into `NaN` comparisons — and §5.2 F5 says a building is visible only if at least one of its units passes. Putting that logic in `lib/filter.ts` keeps components dumb and lets later tasks extend the predicate (beds in 05-08, district/building text in 05-09) without touching a component.

## Current state (preconditions)

- 05-03 is done: `web/src/types.ts` exports `Unit` and `Building`; the committed fixture has units at 45, 50, 58, 60, 65 and 150 sqm.
- `web/src/lib/` does not exist; no filter module, no test helper module.
- The jsdom test setup from 05-02 is green, so a pure `.test.ts` runs.

## Read before you start

- `CLAUDE.md` §3 (table-driven tests for pure functions), §5 (one implementation of a rule; no `any`), §6, §7.
- `plan/02-architecture.md` §5.2 rules F1 (size) and F5 (a building needs ≥1 passing unit); §5.3 lists `lib/filter.ts` as the home.
- `plan/stage-05-web-shell.md` Task 5.3 — the `Filters` shape and the filter case table.
- `plan/90-test-matrix.md` row W-02 — "size bounds inclusive, null = unbounded, invalid ignored (F1)".
- `web/src/types.ts` — the exact unit fields you may read (`sizeSqm`, `priceThb`, `bedrooms`).

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `web/src/lib/filter.ts` | `Filters`, `filterUnits`, `filterBuilding` |
| create | `web/src/lib/filter.test.ts` | the six cases below + a `makeFilters` helper |

Nothing else may change: no component, no hook, no `group.ts`, no fixture edit.

## Test first (red)

Write the test file with one local `makeFilters(overrides)` helper (defaults: both bounds `null`, `beds "any"`, `district "all"`, `building ""`) and one `unit({...})` factory. Build inline units — sizes 45, 58, 65, 150 are enough; do not duplicate the fixture's six units here.

1. `test_size_bounds_are_inclusive_on_both_ends` — bound 45…65 keeps units of size 45 and 65, drops 44 and 66; a single-value bound 65…65 keeps only 65.
2. `test_null_bounds_mean_unbounded` — both null returns all four units in input order; only `sizeMin` set gives an upper-unbounded result; only `sizeMax` set the mirror.
3. `test_non_finite_bounds_are_ignored_not_nan` — `sizeMin: NaN`, `sizeMax: NaN` and `Infinity`/`-Infinity` each behave like the absent bound (all four units), proving the rule is "ignore", not a comparison that drops everything.
4. `test_size_min_above_size_max_returns_empty_without_throwing` — 100…50 returns `[]` (not a throw, not all units), and no `console` call is made.
5. `test_filter_does_not_mutate_its_input` — the input array still has four units and the same object identities afterwards; the result is a different array.
6. `test_filterBuilding_returns_null_when_no_unit_passes` — a building with sizes 58 and 65 under `sizeMin: 100` returns `null`; the same building under `sizeMax: 65` is not null.
7. `test_filterBuilding_keeps_identity_and_only_passing_units` — a building with 45 and 65 under `sizeMax: 50` returns a building with the same `id`, `name`, `district`, `lat`, `lng` and `units` containing only the 45 sqm unit; the `unitCount`/price/size summary fields are **not** recomputed here (05-10 owns that).

Expected red symptom: `Cannot find module './filter'` / `Failed to resolve import './filter'`. If a case fails with a syntax error, fix the test first.

Run: `pnpm -C web test filter` — all seven red for the missing module.

## Implement (green)

- `Filters`, exactly the plan's shape: `sizeMin` and `sizeMax` as `number | null`; `beds` as `"any" | "studio" | 1 | 2 | 3 | "4+"`; `district` as `string | "all"`; `building` as `string`. Export the interface; no extra fields.
- `filterUnits(units: Unit[], f: Filters): Unit[]` — a new array in input order; for now only the size rule is active (`sizeSqm >= sizeMin` when the bound is a finite number, `<= sizeMax` likewise). `null`, `undefined` and any non-finite bound mean "no bound"; do not clamp, coerce strings or convert a bad bound to `NaN`. A contradictory pair produces the empty array naturally; do not special-case it with a throw or a console warning. Leave `beds`, `district` and `building` unread in this task — 05-08 and 05-09 add them with their own tests.
- `filterBuilding(b: Building, f: Filters): Building | null` — `null` when `filterUnits(b.units, f)` is empty, otherwise a shallow copy of the building with `units` replaced by the passing units. Do not mutate `b`, do not touch the dataset, and do not recompute `unitCount`/`priceMin`/`priceMax`/`sizeMin`/`sizeMax` (that is `groupByBuilding`, 05-10, and duplicating it here would give two answers to F5).
- One-sentence docstrings on both functions; no `any`; no React import.

Run: `pnpm -C web test filter` — seven green.

## Refactor / keep it DRY

The size rule exists once, inside `filterUnits`; `filterBuilding` delegates instead of re-implementing the comparison. Bounds are normalised in one local helper (`hasBound`/`inRange`-style) rather than in four `if` branches. Keep the `makeFilters` helper in this test file; `group.test.ts` and `viewport.test.ts` get their own builders rather than a shared module (that is a later refactor if a third caller appears).

## Verification (how to test it)

```text
pnpm -C web test filter
pnpm -C web test --run
pnpm -C web typecheck
pnpm -C web build
make check
```

Expected: 7 passed for the target, 37 for the lane, typecheck 0, build clean, `make check` green. Manual: `grep -n 'NaN\|console' web/src/lib/filter.ts` prints nothing.

## Out of scope (do not do)

- No beds, district or building-text filtering (05-08/05-09); no sorting, grouping or range recomputation (05-10).
- No text parsing of user input, debounce, or string-to-number coercion — the filter bar does that in stage 06.
- No component, hook, `useFilters` state (05-14), no `console.warn`, no dependency.

## Definition of done

- [ ] All seven cases exist in `web/src/lib/filter.test.ts`, named as written, and pass.
- [ ] Red run observed and quoted; no assertion weakened and no test skipped.
- [ ] `pnpm -C web typecheck` clean; lane suite and `make check` green; no `any`.
- [ ] Only the two listed files are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled; exactly one commit on the trunk (`main`) with the message below.

## Commit

```text
05-07: add the inclusive size-bounds filter

- points: 1
- red: filter.test.ts::test_size_bounds_are_inclusive_on_both_ends failed (Cannot find module './filter')
- add Filters, filterUnits with inclusive bounds and ignored non-finite input
- add filterBuilding returning null when no unit passes
- suite: 37 passed
```

Files in the commit: `web/src/lib/filter.ts`, `web/src/lib/filter.test.ts`, and this task file.

## Traps and review notes

- `unit.sizeSqm >= NaN` is `false`, so an unguarded NaN bound silently drops everything; the test in case 3 is the guard against that regression.
- `sizeMin > sizeMax` must yield `[]` without a throw: the UI allows it while the user types, before validation.
- Returning the input array (or mutating it) would let a component accidentally filter the dataset in place; the identity assertion in case 5 catches it.
- Do not recompute building summaries in `filterBuilding`; a reviewer should see no arithmetic on `unitCount`, `priceMin`, `sizeMin` in this file.
- Reviewers: `grep -n 'sizeMin\|sizeMax' web/src/lib/filter.ts` shows the bound normalisation used by one predicate; no duplicated comparison.

## Plan samples are untrusted

The plan gives the `Filters` shape and the case table but no function body. Verify the union syntax and that `null` is the only unbounded signal against the installed TypeScript (`CLAUDE.md` §6); do not paste a snippet. If the sample's `Filters` differs from this task, the task wins for scope; record the deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Deviations | |
