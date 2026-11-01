---
id: 05-08
stage: 05
title: "Bedrooms filter: studio and 4+"
points: 0.5
owner: web
status: todo
depends_on: [05-07]
files:
  - web/src/lib/filter.ts
  - web/src/lib/filter.test.ts
---

# 05-08 · Bedrooms filter: studio and 4+

**Points:** 0.5 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 05-07 · **Unblocks:** 05-09

## Goal

Teach the existing filter predicate the five bedroom values — `any`, `studio` (exactly 0), `1`, `2`, `3` (exact) and `4+` (four or more) — so `filterUnits` implements F2 end to end.

## Why this task exists

Bedrooms are the second filter a renter reaches for, and their semantics are the easiest to get subtly wrong: `Student`/studio must mean zero, not "up to one"; `4+` must include 4, 5 and 6 rather than only 4; and the numeric options must match exactly, so a 2-bed control never shows a 3-bed listing. The `Filters` type already carries the union from 05-07 — this task makes `filterUnits` honour it and pins every edge with a table, while `filterBuilding` keeps delegating so the building-level visibility rule (F5) does not fork.

## Current state (preconditions)

- 05-07 is done: `web/src/lib/filter.ts` exports `Filters`, `filterUnits` (size bounds only) and `filterBuilding` (null when empty); `web/src/lib/filter.test.ts` has seven green cases and the `makeFilters` helper.
- `beds` is present in the `Filters` type but unread by the predicate; the default helper value is `"any"`.
- Inline unit factories in the test file currently vary only `sizeSqm`.

## Read before you start

- `CLAUDE.md` §3 (table-driven pure-function tests), §5 (one implementation, no `any`), §6, §7.
- `plan/02-architecture.md` §5.2 rule F2 — the exact bedroom vocabulary; §5.1 for the beds control.
- `plan/stage-05-web-shell.md` Task 5.3 — the `Filters.beds` union and the "studio matches only 0 / `4+` matches 4,5,6" cases.
- `plan/90-test-matrix.md` row W-03 — "beds exact, Studio=0, 4+ = ≥4".
- `web/src/lib/filter.ts` and `web/src/lib/filter.test.ts` — read the current predicate and the helper before extending either.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/lib/filter.ts` | the bedroom branch inside `filterUnits` |
| modify | `web/src/lib/filter.test.ts` | the four cases below, using the existing helper |

Nothing else may change: no `Filters` reshape, no component, no `group.ts`, no fixture edit.

## Test first (red)

Extend the existing `unit` factory with a `bedrooms` override and build one row per count 0…6. Do not create a second test file and do not add a new helper module.

1. `test_studio_matches_only_zero_bedrooms` — seven units with bedrooms 0…6, `beds: "studio"` returns exactly the 0 unit; `beds: "any"` returns all seven in input order (the no-filter branch, asserted alongside so a stray filter cannot hide here).
2. `test_exact_bedroom_counts_match_only_that_count` — table-driven over `1`, `2`, `3`: each returns exactly the units with that count and never the studio (0) or a larger unit; `beds: 2` does not return 3-bed units.
3. `test_four_plus_matches_four_five_six` — `beds: "4+"` returns the units with 4, 5 and 6 bedrooms, drops 3, and does not throw when a building has no 4+ unit (empty array is valid).
4. `test_bedroom_branch_composes_with_size_bounds` — `beds: "4+"` together with `sizeMin: 100` returns only 4+ units of at least 100 sqm, proving the two rules are ANDed rather than replacing each other.

Expected red symptom: `expected [ {…}, {…}, … ] to deeply equal [ {…} ]` for the studio case (the predicate ignores `beds`, so every unit passes), then the same shape for the exact cases. If a case fails with a TypeError, fix the test first.

Run: `pnpm -C web test filter` — four new cases red, seven old cases green.

## Implement (green)

- Keep the `Filters.beds` union exactly as declared (`"any" | "studio" | 1 | 2 | 3 | "4+"`); do not widen it to `number` or `string`.
- In `filterUnits`, add one bedroom branch: `"any"` matches everything (no work), `"studio"` matches `bedrooms === 0`, `"4+"` matches `bedrooms >= 4`, and the numbers `1`/`2`/`3` match `bedrooms === value`. The branch is ANDed with the existing size rule; unit order and immutability rules from 05-07 are unchanged.
- Do not touch `filterBuilding`: it must keep delegating to `filterUnits` so a building disappears exactly when its last passing unit does.
- Keep the single predicate: no second `matchesBeds` exported, no per-option callback table with one caller. A docstring sentence on `filterUnits` now lists both rules.

Run: `pnpm -C web test filter` — eleven green.

## Refactor / keep it DRY

One bedroom comparison inside the predicate; the size and bedroom rules stay two pure conditions combined in one place. The test table lives beside the existing size table and reuses the same factories. If the predicate starts to need option-specific code, stop and simplify instead of adding exports.

## Verification (how to test it)

```text
pnpm -C web test filter
pnpm -C web test --run
pnpm -C web typecheck
pnpm -C web build
make check
```

Expected: 11 passed for the target, 41 for the lane, typecheck 0, build clean, `make check` green. Manual: `grep -n 'beds' web/src/lib/filter.ts` shows one comparison block, not a copy in `filterBuilding`.

## Out of scope (do not do)

- No district or building-text filtering (05-09); no bathroom filter, no floor filter, no price filter.
- No UI control, parsing or debounce (stage 06); no `useFilters` hook (05-14); no sorting.
- No change to the `Filters` type, the size rule or `filterBuilding`'s contract.

## Definition of done

- [ ] All four cases exist in `web/src/lib/filter.test.ts`, named as written, and pass.
- [ ] Red run observed and quoted; the seven 05-07 cases still pass unchanged.
- [ ] `pnpm -C web typecheck` clean; lane suite and `make check` green; no `any`.
- [ ] Only the two listed files are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled; exactly one commit on the trunk (`main`) with the message below.

## Commit

```text
05-08: filter units by bedrooms, studio and 4+

- points: 0.5
- red: filter.test.ts::test_studio_matches_only_zero_bedrooms failed (expected [0,1,2,3] to deeply equal [0])
- match studio to zero, exact counts to 1/2/3 and 4+ to four or more
- and the bedroom rule with the size bounds in the single predicate
- suite: 41 passed
```

Files in the commit: `web/src/lib/filter.ts`, `web/src/lib/filter.test.ts`, and this task file.

## Traps and review notes

- `studio` matching `bedrooms <= 1` is the classic bug; the studio case must keep a 1-bed unit and drop it.
- `4+` implemented as `bedrooms === 4` passes a naive test but fails the 5/6 rows; the table must contain all three.
- Do not accept the string `"4"` or a numeric `4` for the `"4+"` option: the union is exact, and the UI maps its own value.
- Keep `"any"` a true no-op: a `beds` branch that filters on a sentinel number breaks the default filter state.
- Reviewers: `grep -n 'studio\|4+' web/src/lib/filter.ts` shows each literal once; `filterBuilding` still calls `filterUnits` and contains no bedroom comparison.

## Plan samples are untrusted

The plan gives the option list and the case table, no body. Verify the union's narrowing behaviour (`"4+"` vs `4`) under the installed TypeScript and keep the options exactly as §5.2 words them (`CLAUDE.md` §6). If the plan's wording suggests another option set, this task wins; record the deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Deviations | |
