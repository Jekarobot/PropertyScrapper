---
id: 06-12
stage: 06
title: Empty and error states with reset and retry
points: 0.5
owner: web
status: todo
depends_on: [06-09, 06-05]
files:
  - web/src/App.tsx
  - web/src/App.test.tsx
  - web/src/components/EmptyState.tsx
  - web/src/components/EmptyState.test.tsx
---

# 06-12 · Empty and error states with reset and retry
**Points:** 0.5 · **Stage:** 06 — Filters and results list · **Owner lane:** web · **Depends on:** 06-09, 06-05 · **Unblocks:** nothing

## Goal
Complete the two dead ends: filters that match nothing show the empty state whose action restores the full list, and a failed dataset load shows a full-panel `EmptyState` with a Retry that reloads safely under rapid clicks.

## Why this task exists
The list already renders an empty state and passes errors through, but nobody owns the actions: a visitor who over-filters sees "No buildings in this area" with no way back, and a failed load leaves a blank panel. Both are one-line wiring
mistakes away from shipping, and both are the states a first-time visitor hits on a bad network or with a narrow filter. Retry also has a concurrency hazard: a double click on a slow reload must not stack requests or throw into an unmounted
component.

## Current state (preconditions)
- 06-09 done: `ResultsList` renders the empty state with a `Clear filters` action and renders nothing when `status` is `"error"`; `App` mounts it with `onClearFilters`.
- 06-05 done: `App` owns `filters`, `selectedId` and the filtered buildings; `onReset` already clears both.
- 05-20 done: `EmptyState` takes `{ title, hint, action? }`; 05-13 done: `useDataset` exposes `{ status, data, error, reload }` and aborts the in-flight load on unmount.
- `App.test.tsx` has the 06-05/06-09 cases, including a stubbed `loadDataset` fetch.

## Read before you start
- `plan/stage-06-filters.md` Task 6.4 items 2 and 4.
- `plan/02-architecture.md` §5.1 (empty state wording) and §5.2 row F9 (deselect clears the panel — the same "no ghost" concern applies when filters empty the list).
- `plan/90-test-matrix.md` row W-14 (empty/loading/error states).
- `web/src/state/useDataset.ts`, `web/src/state/useSelection.ts`, `web/src/components/EmptyState.tsx` — the existing reload/abort behaviour and the action prop.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/App.tsx` | clear-filters callback, error panel with Retry |
| modify | `web/src/App.test.tsx` | the five cases below |
| modify (only if needed) | `web/src/components/EmptyState.tsx` | nothing unless the tests demand it |
| modify (only if needed) | `web/src/components/EmptyState.test.tsx` | only for a contract change you actually made |

Nothing else may change: no `ResultsList`, no `useDataset`, no `lib/` edits.

## Test first (red)
1. `test_zero_matching_units_with_a_non_empty_dataset_shows_the_empty_state` — with the six-unit fixture, set the building filter to `zzz` (or `sizeMin` 500): the header reads `0 buildings · 0 units`, no rows render, and the empty state title
   `No buildings in this area` appears.
2. `test_clear_filters_from_the_empty_state_restores_the_full_list` — click `Clear filters`: three rows, `6 found`, the empty state gone, and no row carries `aria-current` (the selected building did not survive the round trip).
3. `test_loader_error_shows_a_full_panel_empty_state_with_retry` — the stubbed loader rejects: the results panel shows an `EmptyState` whose title names the load failure and whose action is `Retry`; no list and no skeleton remain.
4. `test_retry_calls_reload_once_per_click` — with the loader still failing, one click calls the loader exactly once more; a second click adds exactly one.
5. `test_rapid_retry_clicks_do_not_crash` — five clicks in a row on a pending load: no unhandled rejection, no console error, the panel stays coherent (error until a load settles), and when the retry resolves the list renders and the error
   panel is gone.

Expected red: `Unable to find role="button" and name "Retry"`. Run `pnpm -C web test --run App`, quote one line.

## Implement (green)
- Clear filters: `App` passes a callback that calls the same reset path as the bar's Reset — `reset()` plus `select(null)` — so the empty-state action and the Reset button cannot diverge (one function, two entry points).
- Error: when `status` is `"error"`, render the `EmptyState` in place of the results region with a title naming the failure (e.g. `Could not load the dataset`), the loader's message as the hint if it is safe to show, and a `Retry` action
  calling `reload`. Keep the filter bar and header visible; do not clear filters.
- Rapid clicks: `reload` must be safe to call while a load is pending — the hook aborts the previous request; if it does not, disable `Retry` while `status` is `"loading"` rather than editing the hook. Never leave a stale rejection unhandled.
- `EmptyState` already renders `action`; reuse it. Change the component only if a test demands a second prop, and keep the prop shape minimal.

Run `pnpm -C web test --run App` — the new five green, earlier cases unchanged.

## Refactor / keep it DRY
One reset function shared by the bar and the empty state; one error panel. Do not add a second `EmptyState` variant with duplicated markup. The empty-state strings live in one place each (list vs panel).

## Verification (how to test it)
```text
pnpm -C web test --run App
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: all `App` cases green including the 06-05 ones; suite and typecheck clean; `make check` exits 0.

## Out of scope (do not do)
- Retry/backoff policy, toasts, offline detection, service workers.
- Changing `useDataset`'s contract, the loader, or the dataset schema.
- The map/viewport empty states (stage 07) and the top freshness banner (05-19).
- Preserving the selection across a reset (it must be cleared).

## Definition of done
- [ ] Five cases exist, named as written, and pass; no earlier `App` case was edited.
- [ ] Red run observed and quoted; the rapid-click case asserts absence of unhandled rejections.
- [ ] Suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the files above (an untouched `EmptyState` is fine).
- [ ] `status: done`; `## Result` filled; one commit on `main`.

## Commit
```text
06-12: add empty and error states with reset and retry

- points: 0.5
- red: test_loader_error_shows_a_full_panel_empty_state_with_retry failed (Unable to find role="button" and name "Retry")
- wire Clear filters to the same reset as the bar, clearing the selection
- render a full-panel EmptyState with Retry on loader error; rapid clicks stay safe
- suite: N passed
```

Commit this task alone, with the listed files only, on the trunk (`main`) — one task = one
commit, no branch, no push.

## Traps and review notes
- Wiring `Clear filters` to `reset()` only leaves a ghost selection; the same handler as Reset avoids two behaviours.
- Showing the loader's raw error text can leak the fetch URL; show a stable message and keep details out of the panel.
- A `Retry` that stays enabled while the request is in flight stacks requests; either the hook aborts or the button is disabled — pick one and test it.
- Setting state after unmount during a retry is the classic crash here; the hook's abort signal is the mechanism, do not suppress the warning.
- Reviewer greps `App.tsx` for two different reset implementations, and for `reload` calls not originating from a click.

## Plan samples are untrusted
The plan names the states and the strings, not the wiring. Verify `useDataset`'s actual return shape and the abort behaviour against `web/src/state/useDataset.ts` (`CLAUDE.md` §6) before assuming `reload` is idempotent; record deviations
under `## Decisions`.

## Decisions
## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
