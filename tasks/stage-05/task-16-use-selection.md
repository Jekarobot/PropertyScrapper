---
id: 05-16
stage: 05
title: Selection hook that drops buildings leaving the filtered set
points: 0.5
owner: web
status: todo
depends_on: [05-10]
files:
  - web/src/state/useSelection.ts
  - web/src/state/useSelection.test.ts
---
# 05-16 · Selection hook that drops buildings leaving the filtered set
**Points:** 0.5 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 05-10 · **Unblocks:** 05-17
## Goal
Add `useSelection`, which holds the selected building id, resolves it against the current filtered building
list, and clears it when that building is no longer in the list.
## Why this task exists
Rule F9 says one selection spans the map, the results list and the left panel. The trap is that filters and the
map cut change the building list continuously: if the selected building disappears, the panel would keep showing
a building the user can no longer see — a ghost. Resolving the selection against the list the app is actually
rendering, and dropping it when the id is gone, is the single place that answers "which building is open"; both
the panel (07-07) and the map highlight (07-09) read it from here.
## Current state (preconditions)
- 05-10 done: `web/src/lib/group.ts` exports `groupByBuilding` returning `Building[]` with recomputed ranges, and
  every building has a stable `id` (e.g. `p00348`). `web/src/types.ts` (05-03) has the `Building` type.
- 05-01/05-02 done: Vitest + jsdom + `@testing-library/react` wired, `pnpm -C web test --run` green.
  `web/src/state/` exists from 05-13; no `useSelection.ts` yet.
## Read before you start
- `CLAUDE.md` §3 (TDD), §5 (no `any` in `web/src/state/**`; components stay dumb).
- `plan/stage-05-web-shell.md` Task 5.4 — `useSelection` returns `selectedId`, `select`, `selected`, and the note
  that a stale selection must be dropped.
- `plan/02-architecture.md` §5.2 rule F9 — selecting, deselecting, and what clearing means.
- `web/src/lib/group.ts` — the `Building` shape and the guarantee that ids are stable across renders.
## Files to touch
| Action | Path | What goes there |
|---|---|---|
| create | `web/src/state/useSelection.ts` | the hook |
| create | `web/src/state/useSelection.test.ts` | the five cases below |

Only these two files change; report anything else you believe is needed.
## Test first (red)
Use `renderHook` with `initialProps` and the installed RTL `rerender` to change the building list between
renders; build tiny building objects inline (test data duplication is fine, logic duplication is not).

1. `useSelection.test.ts::test_initial_selection_is_empty` — first render → `selectedId` `null` and `selected`
   `null`.
2. `useSelection.test.ts::test_select_stores_the_id_and_resolves_the_building` — `select("p00348")` → `selectedId`
   `"p00348"` and `selected` is the building object with that id from the list passed in.
3. `useSelection.test.ts::test_select_null_clears_the_selection` — after selecting, `select(null)` → `selectedId`
   `null`, `selected` `null` (deselecting by clicking empty map is 07-03's caller, not this task).
4. `useSelection.test.ts::test_selection_is_cleared_when_the_building_leaves_the_filtered_set` — select `p00348`,
   then re-render with a list that no longer contains it → `selectedId` `null` and `selected` `null`; the panel
   therefore has nothing to show.
5. `useSelection.test.ts::test_selection_survives_a_rerender_with_an_equal_building_list` — select `p00348`, then
   re-render with a **new array** holding the same buildings → the selection is kept. This proves the cleanup is
   keyed on membership, not on array identity, so a filter recompute that changes nothing does not close the panel.

Expected red: `Failed to resolve import "./useSelection" from "src/state/useSelection.test.ts"` for every case;
run `pnpm -C web test --run useSelection` and copy one failure line.
## Implement (green)
Create `web/src/state/useSelection.ts` with one hook:

- `useSelection(buildings: Building[])` → `{ selectedId, select, selected }`; `selectedId` is `string | null`,
  `select(id: string | null): void`, `selected` is `Building | null`.
- `selected` is derived, never stored: the building in the passed list whose `id` equals `selectedId`, otherwise
  `null`. Memoise on the list and the id.
- An effect clears `selectedId` when it is non-null and no building in the current list has that id (case 4).
- `select` keeps a stable identity across renders. One-sentence docstring: it holds the selected id, resolves it
  against the given list, and drops it when the building is gone.
- No search over the whole dataset, no panel, no map, no scrolling, no history.

Run: `pnpm -C web test --run useSelection` — five green.
## Refactor / keep it DRY
The "resolve id against the current list" lookup lives here once; `App.tsx` must not repeat the `find`. Do not
keep a second copy of the selected `Building` object in state — the id plus the current list is the truth, and
that is exactly why a ghost cannot appear. No duplicated membership check between the effect and the derived
value beyond the one comparison each needs.
## Verification (how to test it)
```text
pnpm -C web test --run useSelection
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: 5 passed, suite green, `tsc --noEmit` clean, `make check` green.
## Out of scope (do not do)
- The building panel, map bubbles or their highlight — 07-07 and 07-09.
- Expand/collapse of a row, scroll-into-view, focus management, or a "previous selection" stack.
- Storing the `Building` object itself, or a `selectedIndex` instead of an id.
- URL/history sync, multi-select, or selection of units (only buildings are selectable, F9).
## Definition of done
- [ ] All five cases exist, named as written, and pass.
- [ ] Red run observed and quoted in the commit body; target tests + lane suite + typecheck green.
- [ ] No `any`; `selected` derived, not stored; only the two listed files plus this task file are modified.
- [ ] `status: done`, `## Result` filled; exactly one trunk commit with the message below.
## Commit
Message (first line exactly, then body):

```text
05-16: add building selection hook

- points: 0.5
- red: test_initial_selection_is_empty failed (Failed to resolve import "./useSelection")
- add useSelection over the filtered building list: select, derived selected, cleared when gone
- membership check keyed on ids so an equal list keeps the selection
- suite: <N> passed
```

Commit only the listed files plus this task file.
## Traps and review notes
- Storing the `Building` object in state (or keeping `selected` after a list change) is exactly the ghost-panel
  bug: the panel then renders a building that is not in the visible list.
- Keying the cleanup on the array reference instead of the ids closes the panel on every unrelated re-render; case
  5 catches it.
- Reviewers: grep `web/src` for `find(` on buildings — it may appear once, in this hook, and once per its test.
## Plan samples are untrusted
`plan/stage-05-web-shell.md` Task 5.4 gives the signature as `useSelection(ds)`; the contract that makes the
ghost impossible is "resolve against the filtered list", so this task takes the current building list. Apply
`CLAUDE.md` §6 and record the disagreement under `## Decisions`; add no dependency.
## Decisions

## Result
| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
