---
id: 06-10
stage: 06
title: Counts header and partial-data warning
points: 1
owner: web
status: todo
depends_on: [06-09]
files:
  - web/src/components/ResultsList.tsx
  - web/src/components/ResultsList.test.tsx
---

# 06-10 · Counts header and partial-data warning
**Points:** 1 · **Stage:** 06 — Filters and results list · **Owner lane:** web · **Depends on:** 06-09 · **Unblocks:** nothing

## Goal
Give the results list a header that states the size of the current set — `N buildings · M units` with correct singular forms — and an inline `role="status"` warning when the dataset is partial.

## Why this task exists
The list answers "what is here", not "how much is here". Without a header a visitor cannot tell whether the filters narrowed the set or the dataset itself is small, and the map (stage 07) will show a different subset again. The warning
matters more: `stats.partial` means the crawl was aborted or a page failed, so the numbers understate reality, and the product must never present a truncated dataset as complete (F11, and the "partial crawl" honesty rule). Both are
display-only: the component receives the already filtered, already viewport-cut buildings and must not recount anything.

## Current state (preconditions)
- 06-09 done: `ResultsList` renders rows in the given order, the three-row skeleton, the empty state and the error pass-through; it has props `buildings`, `selectedId`, `onSelect`, `status`, `onClearFilters`.
- `App` mounts the list with the filtered/grouped buildings from 06-05.
- 05-03 done: the `Dataset` type has `stats.partial: boolean`; the sample fixture sets it explicitly.
- The pattern for a status region with a `title` already exists in `StatusBar` (05-19) — reuse its wording rules, not its markup.

## Read before you start
- `plan/stage-06-filters.md` Task 6.4 items 1 and 3.
- `plan/02-architecture.md` §5.2 row F11 (header warns when `stats.partial`; the status bar shows `generatedAt`) and §5.1 (header line `header 56px` is the app chrome — this header is the list's, keep the two visually distinct).
- `plan/90-test-matrix.md` rows W-10 (freshness/partial) and W-14; `plan/02-architecture.md` §3.3 (`stats.partial = true` when the crawl was aborted or a page failed).
- `web/src/components/StatusBar.tsx` — the existing partial banner; do not duplicate its logic, and do not remove it.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/components/ResultsList.tsx` | the counts header and the partial warning |
| modify | `web/src/components/ResultsList.test.tsx` | the six cases below |

Nothing else may change: no `App.tsx`, no `StatusBar`, no `EmptyState`.

## Test first (red)
1. `test_header_shows_building_and_unit_counts` — three buildings summing to six units render `3 buildings · 6 units` (exact string with the middle dot and a space on each side).
2. `test_header_uses_singular_forms_for_one` — one building with one unit renders `1 building · 1 unit`, never `1 buildings` or `1 units`.
3. `test_header_mixes_singular_and_plural` — one building with six units renders `1 building · 6 units`.
4. `test_partial_data_shows_an_inline_status_warning` — with `partial` true the list shows an element with `role="status"` whose text is exactly `Data may be incomplete (partial crawl)` and whose `title` attribute is non-empty and explains the
   cause (crawl aborted or a page failed); it sits above the rows.
5. `test_complete_data_shows_no_warning` — with `partial` false there is no status element and no `incomplete` text.
6. `test_counts_reflect_the_given_set_not_the_dataset` — passing two buildings totalling four units renders `2 buildings · 4 units` even though the fixture dataset holds six units: the header counts the props it received, not the dataset, not
   a stored total.

Expected red: `Unable to find text: 3 buildings · 6 units`. Run `pnpm -C web test --run ResultsList`, quote one line.

## Implement (green)
- Props: add `partial: boolean` (required; `App` passes `dataset.stats.partial`). Do not add a `stats` object — one boolean is the contract (YAGNI).
- Counts: `buildings.length` for the first number, the sum of `unitCount` for the second. No other derivation: the buildings are already recomputed, so `unitCount` is already the filtered count. Zero renders `0 buildings · 0 units` alongside
  the empty state.
- Header: one element (a heading is appropriate) with the exact string shape above; place it above the warning and the rows, inside the same named results container.
- Warning: `role="status"`, exact text, and a `title` explaining that the last crawl was incomplete and totals may be understated. Style it inline (muted text, accent border) and do not make it a second page banner — 05-19 already owns the
  top banner; both may coexist.
- Keep loading and empty states working: the header may render during loading (counts of an empty array read `0 buildings · 0 units`) but must not announce them as final; do not add a second live region.

Run `pnpm -C web test --run ResultsList` — thirteen green.

## Refactor / keep it DRY
One pluralisation rule. If `countLabel`-style singular/plural logic already exists in `lib/format.ts`, extend it with a tested helper rather than writing a third inline ternary in the component — but do not edit `lib/` from this task; if a
helper is missing, keep one local function with a comment and report it. Reuse the existing status-bar wording for the explanation, not the component.

## Verification (how to test it)
```text
pnpm -C web test --run ResultsList
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: 13 passed in the target file; suite and typecheck clean; `make check` exits 0.

## Out of scope (do not do)
- `StatusBar` changes, `generatedAt` formatting, the top freshness banner (05-19/W-10 already covers it).
- Per-row counts (already in `BuildingRow`), totals from `dataset.stats`, page counts, crawl diagnostics.
- Sorting, grouping, filtering or viewport logic in the header.
- A dismissible warning, a toast, or a modal.

## Definition of done
- [ ] Six cases exist, named as written, and pass; 06-09 cases still pass.
- [ ] Red run observed and quoted; the exact separator string is asserted, not a regex.
- [ ] Web suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the two files above.
- [ ] `status: done`; `## Result` filled; one commit on `main`.

## Commit
```text
06-10: add results counts header and partial warning

- points: 1
- red: test_header_shows_building_and_unit_counts failed (Unable to find text: 3 buildings · 6 units)
- add the N buildings · M units header with singular and plural forms
- add an inline role=status warning when stats.partial is true
- suite: 13 passed
```

Commit this task alone, with the listed files only, on the trunk (`main`) — one task = one
commit, no branch, no push.

## Traps and review notes
- Counting `dataset` totals instead of the received buildings shows numbers the list does not contain — case 6 exists for exactly that regression.
- `1 buildings`/`0 unit` slips through any test that only checks the plural case.
- A `role="status"` region that remounts on every render re-announces the warning on each keystroke; keep it inside the stable header container.
- Two partial banners (top and list) are acceptable; two live regions announcing the same thing are not — check the accessible tree, not just the DOM.
- Reviewer greps for `stats` in `ResultsList.tsx` (only the boolean prop may appear) and for `toLocaleString` on the counts.

## Plan samples are untrusted
The plan gives the strings, not the markup, and predates the 05-19 status bar. Verify the `Dataset.stats` shape and the existing banner's wording against `web/src/types.ts` and `StatusBar.tsx` (`CLAUDE.md` §6); record deviations under
`## Decisions`.

## Decisions
## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
