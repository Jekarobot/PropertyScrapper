---
id: 06-03
stage: 06
title: Building search input with a capped datalist
points: 1
owner: web
status: todo
depends_on: [06-02]
files:
  - web/src/components/FilterBar.tsx
  - web/src/components/FilterBar.test.tsx
---

# 06-03 · Building search input with a capped datalist
**Points:** 1 · **Stage:** 06 — Filters and results list · **Owner lane:** web · **Depends on:** 06-02 · **Unblocks:** 06-04

## Goal
Add the free-text building field: a labelled text input with a `datalist` of at most 50 suggestions, a 200 ms debounce, trimmed reporting, and Enter doing nothing.

## Why this task exists
Building names are open-ended, so this is the one filter the visitor types. Typing must be cheap (one `onChange` per pause, not per keystroke) and the suggestion list must stay small — the real dataset has about a thousand buildings and a
`datalist` with all of them makes the native popup unusable. The cap belongs in the component because it is a presentation limit, while the value itself is only trimmed here: case-insensitive matching is F4 in `lib/filter.ts`, not in the
bar. Enter must not submit anything, because there is no form and the list already updates live.

## Current state (preconditions)
- 06-01 and 06-02 done: `FilterBar` renders size inputs and both selects; the debounce helper and the render helper in `FilterBar.test.tsx` exist and are reused, not duplicated.
- `FilterBarProps` already declares `buildingOptions: string[]` and `filters.building: string`.
- The bar is still not mounted in `App.tsx`.

## Read before you start
- `plan/stage-06-filters.md` Task 6.1 item 7 (datalist ≤ 50, debounced, Enter is a no-op) and the layout line `Building [____________]`.
- `plan/02-architecture.md` §5.2 row F4 — trimmed, case-insensitive substring on `building.name`, 200 ms debounce; the matching itself is not this task.
- `plan/90-test-matrix.md` row W-11 (filter bar accessible names, debounce, validation).
- `web/src/components/FilterBar.tsx` — the 06-01 debounce helper and the draft-state pattern to reuse.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/components/FilterBar.tsx` | the building input, its datalist and its draft state |
| modify | `web/src/components/FilterBar.test.tsx` | the six cases below |

Nothing else may change: no `App.tsx`, no new module, no interface change, no debounce rewrite.

## Test first (red)
1. `test_building_input_has_an_accessible_name_and_a_linked_datalist` — `getByLabelText("Building")` resolves; the input's `list` attribute names a `datalist` in the bar, and with three `buildingOptions` that datalist holds exactly three
   options with those exact texts.
2. `test_datalist_is_capped_at_fifty_options_in_prop_order` — passing 200 options yields exactly 50 `option` elements whose values are the first 50 props, in order. The cap lives in the component, so a longer prop is legal input, not a bug.
3. `test_typing_is_debounced_and_trimmed_before_reporting` — type `" Bangkok "`: no `onChange` before 200 ms, then exactly one `("building", "Bangkok")` — trimmed, case preserved.
4. `test_rapid_typing_produces_exactly_one_change_with_the_final_text` — type `the bangkok sathorn` keystroke by keystroke inside the debounce window: one `onChange` after the pause, carrying the full final text; none for the intermediate
   keystrokes.
5. `test_clearing_reports_an_empty_string_immediately` — clearing a prefilled value reports `("building", "")` at once; the empty string is the "no filter" value here, not `null`.
6. `test_enter_does_not_submit_or_report_anything` — pressing Enter in the field submits nothing (no form), and adds no extra `onChange` beyond the pending debounced one.

Expected red: `Unable to find a label with the text of: Building`. Run `pnpm -C web test --run FilterBar`, quote one failure line.

## Implement (green)
- A text input with accessible name `Building` (label text, not a placeholder — a placeholder may exist but must not be the only name), controlled by a local draft that follows `filters.building`.
- `datalist` with a stable id (generated once per mount, not a hardcoded global id) and at most 50 `option` values taken from the front of `buildingOptions`. No relevance ranking here: the prop order is the order; `App` decides what the 50
  best options are, the component only enforces the ceiling.
- Reporting: reuse the 200 ms debounce helper from 06-01. Commit the trimmed text; if trimming changes nothing, still report once. Never lowercase, never strip inner spaces, never split on commas.
- Clearing reports `""` immediately and cancels the pending timer, matching the size-pair rule.
- Enter: nothing. No submit handler, no keydown interception that fabricates a change.

Run `pnpm -C web test --run FilterBar` — nineteen green.

## Refactor / keep it DRY
The debounce timer and the draft-following pattern come from 06-01; the building field must not introduce a second timer implementation. The 50 constant is named once. No new helper module: the file list does not allow it.

## Verification (how to test it)
```text
pnpm -C web test --run FilterBar
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: 19 passed in the target file; suite and typecheck clean; `make check` exits 0.

## Out of scope (do not do)
- Reset, live count, group name, tab order (06-04); wiring into `App` (06-05).
- Fuzzy matching, relevance ranking, tokenising, case folding, debounce values other than 200 ms.
- Fetching suggestions, a custom combobox, keyboard-navigable popup logic.

## Definition of done
- [ ] Six cases exist, named as written, and pass; earlier `FilterBar` cases still pass.
- [ ] Red run observed and quoted; the debounce assertion counts calls, it is not weakened to "called".
- [ ] Web suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the two files above.
- [ ] `status: done`; `## Result` filled; one commit on `main`.

## Commit
```text
06-03: add building search with a capped datalist

- points: 1
- red: test_building_input_has_an_accessible_name_and_a_linked_datalist failed (Unable to find a label with the text of: Building)
- add the Building input, a 50-option datalist cap and trimmed debounced reporting
- clearing reports an empty string; Enter is a no-op
- suite: 19 passed
```

Commit this task alone, with the listed files only, on the trunk (`main`) — one task = one
commit, no branch, no push.

## Traps and review notes
- Naming the field via `placeholder="search…"` fails 06-04 and the a11y requirement; use a label.
- Lowercasing here looks harmless and silently duplicates F4; the test for case preservation exists to catch it.
- Reporting the untrimmed value makes `" Bangkok "` a filter that matches nothing after F4 trims it again — trim exactly once, on commit.
- A hardcoded `datalist` id collides when two bars are mounted in one test file; generate it.
- Reviewer checks the cap is applied to the rendered options, not to a copy of the array that later grows, and that no `form`/`onSubmit` appeared.

## Plan samples are untrusted
The plan gives the prop name and the ceiling, no markup. Check `datalist`/`list` attribute support in the installed React typings and jsdom before relying on it (`CLAUDE.md` §6); record deviations under `## Decisions`.

## Decisions
## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
