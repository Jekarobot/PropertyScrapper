---
id: 06-09
stage: 06
title: Results list with loading and empty states
points: 1
owner: web
status: todo
depends_on: [06-06, 05-20]
files:
  - web/src/components/ResultsList.tsx
  - web/src/components/ResultsList.test.tsx
  - web/src/App.tsx
  - web/src/App.test.tsx
---

# 06-09 · Results list with loading and empty states
**Points:** 1 · **Stage:** 06 — Filters and results list · **Owner lane:** web · **Depends on:** 06-06, 05-20 · **Unblocks:** 06-10, 06-11, 06-12

## Goal
Create `ResultsList` — building rows in the given order with list semantics, a three-row loading skeleton, and `EmptyState` with a "Clear filters" action — and mount it in `App`, deleting the placeholder list.

## Why this task exists
Rows exist but nothing owns them: no container decides order, loading, or emptiness, and `App` still renders the stage-05 placeholder. The list is also where the app answers "nothing matched" — with the sample dataset that is reachable in
one filter, and without the action the visitor is stuck. The component is deliberately a presenter: it receives the already filtered, already grouped buildings and renders them, so there is exactly one place (`App`) where state becomes
rows. 06-10 adds the counts header here, 06-11 measures this list, 06-12 builds on the states.

## Current state (preconditions)
- 06-06/06-08 done: `BuildingRow` renders name/district/counts/ranges, selects, expands its units.
- 05-20 done: `web/src/components/EmptyState.tsx` takes `{ title, hint, action? }` and renders the action as a button; it is already used by the stage-05 error path.
- 05-18 done: `App` renders a placeholder list (one item per building) inside the results section.
- 06-05 done: `App` derives the filtered, grouped buildings; its `App.test.tsx` cases query list items by role and visible text.
- `web/src/components/ResultsList.tsx` and its test do **not** exist.

## Read before you start
- `plan/stage-06-filters.md` Task 6.3 items 5 and the row-anatomy preamble (`role="row"` inside `role="list"`), plus `plan/02-architecture.md` §5.1 empty state ("No buildings in this area — zoom out or clear filters" with a reset button).
- `plan/90-test-matrix.md` rows W-14 (empty/loading/error states) and W-09 (selection).
- `web/src/components/EmptyState.tsx` and `web/src/App.tsx` — the props to reuse and the placeholder to delete.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `web/src/components/ResultsList.tsx` | the list, skeleton and empty state |
| create | `web/src/components/ResultsList.test.tsx` | the seven cases below |
| modify | `web/src/App.tsx` | replace the placeholder with `ResultsList` |
| modify | `web/src/App.test.tsx` | keep 06-05 cases green; assert rows come from `ResultsList` |

Nothing else may change: no `BuildingRow`, `UnitRow` or `EmptyState` internals, no `lib/` edits.

## Test first (red)
1. `test_renders_one_list_item_per_building_in_the_given_order` — three buildings in the order A, B, C produce three list items whose building names appear in that order (assert the sequence, not a set).
2. `test_list_has_an_accessible_name_and_native_list_semantics` — `getByRole("list", { name: /results/i })` resolves and `getAllByRole("listitem")` counts the buildings. The component carries a one-line comment documenting the choice: native
   `ul`/`li` instead of the plan's `role="row"`, because `row` outside a `table`/`grid` is invalid ARIA and breaks screen-reader table navigation.
3. `test_loading_shows_three_skeleton_rows_and_the_busy_state` — with `status` `"loading"` the list contains exactly three placeholder rows, the container has `aria-busy="true"`, and no building name or `view` link is rendered.
4. `test_empty_result_shows_the_empty_state_and_clear_filters_action` — `buildings: []`, ready: the `EmptyState` shows the title `No buildings in this area`, and the action button `Clear filters` calls `onClearFilters` exactly once.
5. `test_selected_building_is_marked_and_only_that_one` — with a `selectedId`, exactly one row has `aria-current="true"` and it is the matching building; the component passes the flag through and keeps no selection state of its own.
6. `test_ready_state_is_not_busy_and_shows_no_empty_state` — `status` `"ready"` with buildings: no `aria-busy`, no empty-state text.
7. `test_error_status_renders_neither_rows_nor_the_empty_state` — `status` `"error"`: no list, no empty state here (the full-panel error belongs to 06-12); this keeps two competing messages off the screen.

Expected red: `Failed to resolve import "./ResultsList"`. Run `pnpm -C web test --run ResultsList`, quote one line.

## Implement (green)
Export `ResultsList` with props `{ buildings, selectedId, onSelect, status, onClearFilters }`:

- No business logic: render `buildings` exactly as received — no sorting, filtering, grouping or counting — and never import `lib/filter.ts` or `lib/group.ts`; the component is a presenter only.
- Markup: a `ul` with accessible name `Results` (or a `section` named Results wrapping it — one named container, not two) inside the results region from 05-18; each item wraps `BuildingRow` and passes `selected={building.id === selectedId}`
  and `onSelect`.
- Loading: the same container with `aria-busy="true"` and exactly three non-interactive skeleton rows (grey blocks, `aria-hidden` so screen readers do not read placeholder noise).
- Empty: `EmptyState` with title `No buildings in this area`, a hint mentioning zooming out or clearing filters, and the action labelled `Clear filters` wired to `onClearFilters`.
- Error: render nothing; `App` owns the error panel (06-12).
- In `App.tsx`, replace the placeholder markup with `ResultsList`, pass the derived buildings, `selectedId`, `select`, the dataset status and a clear-filters callback. Delete the placeholder row markup added in 06-05; the 06-05 tests must
  keep passing unchanged.

Run `pnpm -C web test --run ResultsList` then `--run App` — both green.

## Refactor / keep it DRY
One list container and one empty state; `EmptyState` is imported, not re-created. `App` must not compute "is empty" twice (the list decides on an empty array). Remove the placeholder completely — leaving both renderers is the failure this
task is meant to avoid.

## Verification (how to test it)
```text
pnpm -C web test --run ResultsList
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: 7 passed for the list; `App` suite green including the untouched 06-05 cases; typecheck clean.

## Out of scope (do not do)
- The counts header and partial warning (06-10); the perf budget (06-11); error panel and retry (06-12).
- Virtualisation, windowing, grouping inside the list, sorting controls.
- Map/viewport cut (stage 07) — `buildings` arrives already cut.

## Definition of done
- [ ] Seven cases exist, named as written, and pass; every 06-05 `App` case still passes unedited.
- [ ] Red run observed and quoted; the loading case asserts absence of real rows, not only presence of skeletons.
- [ ] Web suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the four files above.
- [ ] `status: done`; `## Result` filled; one commit on `main`.

## Commit
```text
06-09: add the results list with loading and empty states

- points: 1
- red: test_renders_one_list_item_per_building_in_the_given_order failed (Failed to resolve import "./ResultsList")
- add ResultsList with native list semantics, three skeleton rows and the empty state
- mount it in App and delete the 06-05 placeholder rows
- suite: N passed
```

Commit this task alone, with the listed files only, on the trunk (`main`) — one task = one
commit, no branch, no push.

## Traps and review notes
- Keeping the placeholder and the list mounted doubles every row; the 06-05 tests will then fail on duplicate names — delete the placeholder, do not adapt the tests.
- `role="row"` outside a table is invalid ARIA; if you follow the plan's wording, document the deviation in `## Decisions` as the comment also does.
- Skeleton rows that are real focusable elements pollute the Tab order; mark them `aria-hidden` and non-interactive.
- Recomputing "empty" in `App` and in the list gives two sources of truth; the list owns it.
- Reviewer greps `ResultsList.tsx` for `filter`/`group`/`sort`/`reduce` on buildings, and `App.tsx` for leftover placeholder markup.

## Plan samples are untrusted
The plan's row anatomy uses `role="row"`/`role="list"` and mentions a `loading` skeleton without markup. Verify ARIA and testing-library role behaviour in the installed versions (`CLAUDE.md` §6); record the deviation under `## Decisions`.

## Decisions
## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
