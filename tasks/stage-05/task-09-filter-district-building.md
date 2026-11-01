---
id: 05-09
stage: 05
title: District and building-text filters
points: 1
owner: web
status: todo
depends_on: [05-08]
files:
  - web/src/lib/filter.ts
  - web/src/lib/filter.test.ts
---

# 05-09 · District and building-text filters

**Points:** 1 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 05-08 · **Unblocks:** 05-14, 06-03

## Goal

Add the two building-level filters to `filterBuilding`: district exact match (`All` = off) and a trimmed, case-insensitive substring of the building name, with a building dropped when its last unit goes.

## Why this task exists

District and building name are the only filters that live on the building, not on the unit, and §5.2 F3/F4 pin their semantics precisely: district compares exactly (so `Watthana` never leaks `Watthana Ovest` or a lowercase typo), the building box is a case-insensitive substring after trimming, and `All`/empty mean "no filter". F5 then requires the whole building to disappear when no unit survives. These three rules decide which rows and bubbles exist, so they are proven on the committed fixture before any component reads them.

## Current state (preconditions)

- 05-08 is done: `filterUnits` applies size and bedroom rules; `filterBuilding` returns null when the unit list is empty and otherwise copies the building with the passing units.
- `web/src/lib/filter.test.ts` has eleven green cases, the `makeFilters` helper and the unit factory.
- The fixture has three buildings: `p00348` (`Sa Thon`), `premier-thonglor` (`Watthana`) and `no-location` (`Khlong Toei`); only `The Bangkok Sathorn` contains the word `bangkok`.

## Read before you start

- `CLAUDE.md` §3 (exact fixtures, observed red), §5 (one implementation per rule), §6, §7.
- `plan/02-architecture.md` §5.2 rules F3 (district exact, `All` = no filter) and F4 (building substring, case-insensitive, trimmed); §5.1 for the district control.
- `plan/stage-05-web-shell.md` Task 5.3 — the filter case table; `plan/stage-06-filters.md` Task 6.2 for the user-visible expectations (`district=Watthana` keeps only `Premier Thonglor`).
- `plan/90-test-matrix.md` row W-04 and row W-05 (filters before the viewport cut).
- `web/src/types.ts` — note `Unit` has no district or name, which is why this rule lives in `filterBuilding`.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/lib/filter.ts` | building-level checks in `filterBuilding`; a comment on `filterUnits` |
| modify | `web/src/lib/filter.test.ts` | the six cases below, built on the imported fixture |

Nothing else may change: no `Filters` reshape, no `group.ts`, no component, no fixture edit.

## Test first (red)

Add the cases to `web/src/lib/filter.test.ts`. Import the committed fixture for these cases (they are about building identity, so the real three-building shape is the point) and look buildings up by id; do not hand-type a fourth building.

1. `test_district_matches_the_dataset_value_exactly` — `district: "Watthana"` keeps `premier-thonglor` with both units; `"watthana"` (wrong case) and `"Watthan"` (prefix) return `null`; `p00348` under `"Watthana"` is `null`. Pins F3: no case folding, no substring.
2. `test_district_all_means_no_filter` — `district: "all"` with otherwise default filters keeps all three buildings, including `no-location`, with their full unit counts.
3. `test_building_is_a_case_insensitive_trimmed_substring` — `building: "PREMIER"`, `"  premier  "` and `"Thonglor"` each keep `premier-thonglor`; `"bangkok"` keeps only `p00348`; `"zzz"` returns `null`. Pins F4.
4. `test_empty_or_whitespace_building_text_means_no_filter` — `building: ""` and `building: "   "` both keep all three buildings; trimming alone must not become "match nothing".
5. `test_filterBuilding_drops_a_building_whose_units_all_drop_out` — on `premier-thonglor`: `sizeMin: 100` → `null` (its units are 45 and 50 sqm), and `district: "Sa Thon"` → `null` even though its units would pass size. The building is removed, not returned with zero units.
6. `test_building_text_does_not_match_district_or_province` — `building: "Khlong Toei"` (a district value) returns `null`; `building: "Bangkok"` returns only `p00348`, whose name contains it — the province shared by all three buildings must not match, because F4 compares `name` only.

Expected red symptom: `expected null to be null` style failures on the exact-match rows and `expected […] to have length 2` on the substring rows, because `filterBuilding` currently ignores both fields and returns every building. A case that already passes is not testing the new rule.

Run: `pnpm -C web test filter` — six new cases red, eleven old cases green.

## Implement (green)

- `filterBuilding(b, f)`: first the building-level checks, then the unit-level ones. District: `"all"` passes; otherwise `b.district === f.district` (exact, case-sensitive; a `null` district never matches a concrete value). Building text: trim `f.building`; when the trimmed text is non-empty, `b.name.toLowerCase()` must contain `text.toLowerCase()` — no `normalize`, no accent folding, no fuzzy match. A failed building-level check returns `null` immediately without looking at units.
- Then delegate to `filterUnits(b.units, f)`; empty result → `null`; otherwise return a shallow copy with those units, leaving the summary fields untouched (05-10 recomputes them).
- `filterUnits` keeps its unit-level contract and is now documented as such: a `Unit` carries no district or building name, so the two building-level filters are applied by `filterBuilding`. Do not duplicate the district/name comparisons inside `filterUnits`.
- No new exports, no sorting, no logging. Update the one-sentence docstrings to mention the two building-level rules.

Run: `pnpm -C web test filter` — seventeen green.

## Refactor / keep it DRY

Each rule exists once: district equality and name containment are two conditions inside `filterBuilding`; size and bedrooms remain inside `filterUnits`. `filterBuilding` calls `filterUnits` rather than re-implementing F1/F2. The fixture is read through one `buildingById` helper in the test, not copied into local literals.

## Verification (how to test it)

```text
pnpm -C web test filter
pnpm -C web test --run
pnpm -C web typecheck
pnpm -C web build
make check
```

Expected: 17 passed for the target, 47 for the lane, typecheck 0, build clean, `make check` green. Manual: `grep -n 'toLowerCase\|trim' web/src/lib/filter.ts` shows the name rule only, not the district comparison.

## Out of scope (do not do)

- No fuzzy/prefix district matching, no alias list, no accent or Unicode normalisation, no "All districts" dropdown data (stage 06).
- No price filter, no floor/bathroom filter, no sort; no hook or component (05-14, 06-03); no URL sync of filters (F10).
- No change to `filterUnits`' unit-level behaviour, and no recomputation of building summaries here.

## Definition of done

- [ ] All six cases exist in `web/src/lib/filter.test.ts`, named as written, and pass.
- [ ] Red run observed and quoted; the eleven earlier filter cases still pass unchanged.
- [ ] `pnpm -C web typecheck` clean; lane suite and `make check` green; no `any`.
- [ ] Only the two listed files are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled; exactly one commit on the trunk (`main`) with the message below.

## Commit

```text
05-09: add district and building-name filters

- points: 1
- red: filter.test.ts::test_district_matches_the_dataset_value_exactly failed (expected null to be Building)
- match district exactly, name as a trimmed case-insensitive substring
- drop a building when its last unit stops passing
- suite: 47 passed
```

Files in the commit: `web/src/lib/filter.ts`, `web/src/lib/filter.test.ts`, and this task file.

## Traps and review notes

- `Unit` has no district, so district cannot be applied in `filterUnits`; the App composition (05-17) must map `filterBuilding` over `dataset.buildings` rather than filtering a flat unit array. If a later task assumes otherwise, raise it instead of duplicating the rule.
- Lowercasing the district comparison would make `watthana` match; F3 says exact. Lowercasing the name comparison is required by F4 — do not "unify" the two.
- A whitespace-only building box must behave like an empty one; trim before the emptiness check, not after the comparison.
- Returning a building with zero units instead of `null` breaks F5 and shows an empty row in the list.
- Reviewers: `grep -n 'district' web/src/lib/filter.ts` has one comparison; no `includes` on `district`; `filterBuilding` still calls `filterUnits`.

## Plan samples are untrusted

The plan gives rule wording and case names, no body, and its composition sketch filters a flat unit array. Verify against `web/src/types.ts` that `Unit` has no location field; where the sketch and the types disagree, the types win and the deviation is recorded under `## Decisions` (`CLAUDE.md` §6).

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Deviations | |
