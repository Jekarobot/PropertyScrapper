---
id: 05-17
stage: 05
title: App composition with one data flow and retryable load errors
points: 2
owner: web
status: todo
depends_on: [05-13, 05-14, 05-15, 05-16]
files:
  - web/src/App.tsx
  - web/src/App.test.tsx
---
# 05-17 · App composition with one data flow and retryable load errors
**Points:** 2 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 05-13…05-16 · **Unblocks:** 05-18, 06-05, 07-06
## Goal
Wire the four state hooks into the single data flow `App.tsx` owns, so the app renders header totals, one entry
per building, the viewport cut and a retryable error state.
## Why this task exists
Rules F5–F11 (`plan/02-architecture.md` §5.2) only hold if one place turns dataset + filters + map box into the
rendered lists: filters first, then the viewport cut, then a resolved selection. Every later component — results
list (06-09), map (07-02), panel (07-08) — receives precomputed data from here and stays dumb; without it,
stages 06–07 each invent their own wiring.
## Current state (preconditions)
- 05-13…05-16 done: `useDataset`, `useFilters`, `useViewport`, `useSelection` with the contracts in those files;
  read them, do not change them.
- 05-03 done: `web/src/types.ts`, the fixture (6 units / 3 buildings; `slug:no-location` has `lat: null`) and
  `web/src/test/fixtures.ts`; 05-04…05-12 done: loader, `filterUnits` and `filterBuilding` (05-09),
  `groupByBuilding`, `splitByLocation` + `BBox`, formatters.
- `web/src/App.tsx` is the bare scaffold component (05-01 removed the Vite demo); `web/src/App.test.tsx` does not
  exist yet.
## Read before you start
- `CLAUDE.md` §3 (TDD; tests stub `fetch`), §5 (single source of truth in `App.tsx`, components dumb).
- `plan/stage-05-web-shell.md` Task 5.4 — the composition block; `plan/02-architecture.md` §5.2 rules
  F5/F6/F9/F10/F11 and §5.1's header line.
- `plan/90-test-matrix.md` rows W-05, W-09 (hook part is 05-16), W-14; `web/src/lib/*.ts` for the exact
  signatures.
## Files to touch
| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/App.tsx` | the composition, the whole-city box, the rendered content |
| create | `web/src/App.test.tsx` | the five cases below |

Only these two files change; report anything else you believe is needed.
## Test first (red)
Stub `fetch` with the sample dataset in every case. Cases 3–4 exercise the composition directly; 1, 2 and 5
render `<App />`.

1. `App.test.tsx::test_header_shows_dataset_totals` — after load the header contains `6 units` and `3 buildings`,
   taken from `stats`, not recomputed from the filtered list.
2. `App.test.tsx::test_renders_one_entry_per_building` — exactly three entries: both coordinate buildings in the
   results list, `slug:no-location` in a group labelled `No location (1)`; each name appears once.
3. `App.test.tsx::test_filter_before_viewport` (matrix row W-05 name) — filters limited to
   `district: "Khlong Toei"` (only the null-coord building matches) with a box around the coordinate buildings →
   `inView` empty, `noLocation` holds `slug:no-location`; the cut never re-admits a filtered-out building.
4. `App.test.tsx::test_bbox_around_bangkok_moves_the_null_coord_building_to_no_location` — box from the fixture's
   own coordinates (documented `p00348` point 13.7187616 / 100.5202356, ±0.05°; invent no numbers) →
   `slug:no-location` absent from `inView`, present in `noLocation`, dataset order kept.
5. `App.test.tsx::test_error_state_shows_retry_that_refetches` — HTTP 500 → title `Could not load the dataset`,
   hint `Check that the dataset was published, then try again.`, button `Try again`; clicking issues a second
   request and, on success, rows appear and the error is gone.

Expected red: a real assertion failure such as `Unable to find an element with the text: 6 units`. Run
`pnpm -C web test --run App`; a syntax error or typo is not red — fix the test first.
## Implement (green)
Rewrite `web/src/App.tsx` to own the whole flow:

- `useDataset()` for the data; keep the other hooks' results in named objects (`filtersState`, `viewport`,
  `selection`) and read only what is rendered — no unused destructured locals, no lint suppression.
- One module-level pure function, `composeView(dataset, filters, bbox)`, is the only place filtering, grouping
  and cutting happen. Steps in order: map `filterBuilding` over `dataset.buildings` and drop the nulls (district
  and building-name rules live there, because a unit carries neither field); flatten the surviving buildings'
  units; call `groupByBuilding(dataset, units)` so ranges and counts are recomputed from the filtered set; call
  `splitByLocation(buildings, bbox ?? WORLD)`; return `{ units, buildings, inView, noLocation }`. `App` calls it
  in `useMemo` on `[dataset, filters, bbox]`.
- `WORLD` is one module constant, a generous box covering Thailand with a one-line comment (Thailand-only, no
  dateline wrapping — stage plan Task 5.3). `selected` comes from `useSelection(buildings)`; the `find` lives
  there, do not repeat it.
- Render: `header` with the totals and a freshness placeholder; results list of `inView`; a `No location (N)`
  group; a left placeholder panel with the selected name or `No building selected`; the loading and error states
  with case 5's copy.

Run: `pnpm -C web test --run App` — five green.
## Refactor / keep it DRY
`composeView` is the only place that filters, groups and cuts: no component may call `filterBuilding`,
`filterUnits`, `groupByBuilding`, `splitByLocation` or `useDataset` again — stage 06 takes lists and callbacks as props. If
proving a rule needs a DOM, the rule belongs in `lib/`.
## Verification (how to test it)
```text
pnpm -C web test --run App
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: 5 passed, suite green, `tsc --noEmit` clean, `make check` green.
## Out of scope (do not do)
- Filter bar, results-list components, real rows (stage 06); Leaflet or calling `setBBox` (07-04).
- A reusable `EmptyState`: this stage renders the error copy inline, 05-20 extracts it.
- URL sync, routing, a state library, cached composed lists (derived data), or edits to 05-13…05-16, `lib/**`, the
  fixture, the loader.
## Definition of done
- [ ] All five cases exist, named as written, and pass; `test_filter_before_viewport` keeps its matrix name.
- [ ] Red run observed and quoted; target tests + lane suite + typecheck + `make check` green.
- [ ] `App.tsx` has no size/bed/district comparison and no coordinate maths (grep `bedrooms`, `Math.`, `lat`).
- [ ] Only the two listed files plus this task file are modified; `status: done`, `## Result` filled; one trunk
      commit.
## Commit
Message (first line exactly, then body):

```text
05-17: compose app data flow from state hooks

- points: 2
- red: test_header_shows_dataset_totals failed (Unable to find an element with the text: 6 units)
- add composeView: filterBuilding -> flatten -> groupByBuilding -> splitByLocation with bbox ?? WORLD
- render header totals, inView rows, No location group, loading and retryable error states
- suite: <N> passed
```

Commit only the listed files plus this task file.
## Traps and review notes
- Cutting by viewport before filtering, or letting the null-coord building into `inView`, breaks F5/F6 — cases 3–4
  are the guard.
- Recomputing totals from the filtered lists puts filter state into the header; totals come from `dataset.stats`;
  a second `find` for the selection re-creates the ghost-panel bug 05-16 prevents.
- Reviewers: grep `App.tsx` for `filter(`, `.find(`, `Math.` (absent) and `WORLD` (exactly once).
## Plan samples are untrusted
The composition block in Task 5.4 names `dataset`, `filters`, `bbox` and `selectedId` without saying where they
come from, and its selection signature differs from 05-16's. Apply `CLAUDE.md` §6, follow the hooks as
implemented, record any deviation under `## Decisions`. Add no dependency.
## Decisions

## Result
| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
