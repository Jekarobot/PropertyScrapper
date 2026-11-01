---
id: 06-02
stage: 06
title: Bedrooms and district selects in the filter bar
points: 1
owner: web
status: todo
depends_on: [06-01]
files:
  - web/src/components/FilterBar.tsx
  - web/src/components/FilterBar.test.tsx
---

# 06-02 · Bedrooms and district selects in the filter bar
**Points:** 1 · **Stage:** 06 — Filters and results list · **Owner lane:** web · **Depends on:** 06-01 · **Unblocks:** 06-03

## Goal
Add the bedrooms select (Any/Studio/1/2/3/4+) and the district select (All plus the districts prop) to `FilterBar`, reporting `onChange` with the exact `Filters` values.

## Why this task exists
Two of the five filters are finite choices, so they are selects, not text: bedrooms is an exact match with a studio special case and a `4+` bucket (F2), and district is an exact string match with `All` meaning "no filter" (F3). Getting the
option values wrong is invisible until filters silently stop matching — the union in `Filters` mixes numbers and strings — so this task pins the option list, the accessible names, and the value mapping. It also proves the bar stays
controlled: the parent owns the value.

## Current state (preconditions)
- 06-01 done: `FilterBar.tsx` exports `FilterBar`/`FilterBarProps`, renders the size pair, and its test file has one render helper plus seven cases.
- `web/src/state/useFilters.ts` exports `Filters` with `beds: "any" | "studio" | 1 | 2 | 3 | "4+"` and `district: string | "all"`; defaults `"any"` and `"all"`.
- 05-03 done: the sample dataset has districts `Sa Thon`, `Watthana`, `Khlong Toei` (6 units / 3 buildings); `App` will pass them sorted.
- The bar is still not mounted in `App.tsx`.

## Read before you start
- `plan/stage-06-filters.md` Task 6.1 items 5–6 and the layout line (`Beds [Any ▾]`, `District [All ▾]`).
- `plan/02-architecture.md` §5.2 rows F2 (Studio = 0, `4+` = `>= 4`) and F3 (exact district, `All` = none).
- `plan/90-test-matrix.md` rows W-02/W-03/W-04 — those test `lib/filter.ts`; this task proves the component hands it the right values.
- `web/src/state/useFilters.ts` and `web/src/lib/filter.ts` — read the union and the comparison the values must satisfy.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/components/FilterBar.tsx` | the two selects and their value mapping |
| modify | `web/src/components/FilterBar.test.tsx` | the six cases below, reusing the 06-01 render helper |

Nothing else may change: no new file, no interface change, no `App.tsx`.

## Test first (red)
1. `test_bedrooms_select_has_an_accessible_name_and_six_options` — `getByLabelText("Bedrooms")` is a select whose options, in order, read `Any`, `Studio`, `1`, `2`, `3`, `4+`; values are `any`, `studio`, `1`, `2`, `3`, `4+`.
2. `test_bedrooms_select_shows_the_current_filter` — `filters.beds` `"studio"` selects `Studio`; `2` selects the option labelled `2` (a number in `Filters`, a string in the DOM).
3. `test_selecting_four_plus_reports_the_literal` — choosing `4+` calls `onChange("beds", "4+")`, exactly once, with the string.
4. `test_selecting_a_numbered_bedroom_reports_a_number` — choosing `2` calls `onChange("beds", 2)` as a **number**; `toHaveBeenCalledWith("beds", 2)` must not pass for `"2"` (assert the type with `typeof` on the argument).
5. `test_district_select_lists_all_first_then_the_districts_prop` — with `["Khlong Toei", "Sa Thon", "Watthana"]` the options are `All`, then those three in the given order (do not sort inside the component); the `All` option's value is `all`.
6. `test_selecting_a_district_reports_the_value_and_defaults_to_all` — `filters.district` `"all"` shows `All`; choosing `Watthana` calls `onChange("district", "Watthana")`; mounting with `"all"` calls nothing.

Expected red: `Unable to find a label with the text of: Bedrooms` (the select does not exist yet). Run `pnpm -C web test --run FilterBar`, quote one line.

## Implement (green)
- Bedrooms: a `select` with accessible name `Bedrooms` (visible `Beds` label is fine if the accessible name is `Bedrooms`), options in the exact order above, controlled by `filters.beds`. On change, map the DOM string back to the union:
  `any`/`studio` stay strings, `1`/`2`/`3` become numbers, `4+` stays the literal string. That mapping is the whole point of case 4.
- District: a `select` with accessible name `District`, first option `All` (value `all`), then `districts` in prop order, no filtering, no de-duplication, no sorting — `App` owns that. Controlled by `filters.district`.
- Both call the existing `onChange` from 06-01; no other callback is added. No filtering, no fallback when the current value is missing from the options: if `filters.district` is a value not in `districts`, the select shows no selection
  rather than inventing one; do not clamp it.
- Reuse the bar's existing layout and styling tokens; no new class system.

Run `pnpm -C web test --run FilterBar` — thirteen green (seven from 06-01, six here).

## Refactor / keep it DRY
The option list is built once from one local constant (beds) and once from the prop (districts); do not repeat the labels in JSX and in tests-as-logic. No second debounce, no new helper. Keep the size logic untouched.

## Verification (how to test it)
```text
pnpm -C web test --run FilterBar
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: 13 passed in the target file; suite and typecheck clean; `make check` exits 0.

## Out of scope (do not do)
- Building search and datalist (06-03); Reset, live count, group name, tab order (06-04); wiring (06-05).
- Changing `Filters`, `filterUnits`, or the district list source; searchable/combobox district select; multi-select.
- Any `beds` value outside the six options; a "4+" filter implemented in the component.

## Definition of done
- [ ] Six cases exist, named as written, and pass; the 06-01 cases still pass.
- [ ] Red run observed and quoted; no assertion weakened to accept a string for a numbered bedroom.
- [ ] Web suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the two files above.
- [ ] `status: done`; `## Result` filled; one commit on `main`.

## Commit
```text
06-02: add bedrooms and district selects

- points: 1
- red: test_bedrooms_select_has_an_accessible_name_and_six_options failed (Unable to find a label with the text of: Bedrooms)
- add the six bedrooms options with exact Filters values (numbers for 1-3)
- add the district select from props with an All option
- suite: 13 passed
```

Commit this task alone, with the listed files only, on the trunk (`main`) — one task = one
commit, no branch, no push.

## Traps and review notes
- `select.value` is always a string: forgetting the numeric mapping makes `beds=2` never match, because `filter.ts` compares `bedrooms === 2`. This is the trap the `typeof` assertion exists for.
- Do not label the bedrooms select only `Beds` in the accessible tree while the requirement is `Bedrooms`; 06-04 queries by label.
- Do not sort or de-duplicate `districts` here — ordering is decided upstream (03-10) and the bar must reflect it.
- Do not add a placeholder option above `All`, and do not allow an empty district value: `all` is the sentinel.
- Reviewer greps for `Number(` / `parseInt` without the `4+` guard, and for a second copy of the district list.

## Plan samples are untrusted
The plan names the options and the union but gives no markup. Verify option/select APIs against `web/package.json` and the installed React typings (`CLAUDE.md` §6); record deviations under `## Decisions`.

## Decisions
## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
