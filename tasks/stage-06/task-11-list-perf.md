---
id: 06-11
stage: 06
title: Large-list render budget smoke test
points: 0.5
owner: web
status: todo
depends_on: [06-09]
files:
  - web/src/components/ResultsList.test.tsx
---

# 06-11 · Large-list render budget smoke test
**Points:** 0.5 · **Stage:** 06 — Filters and results list · **Owner lane:** web · **Depends on:** 06-09 · **Unblocks:** nothing

## Goal
Add one performance smoke case to `ResultsList.test.tsx`: 200 generated buildings render in under the CI budget of 300 ms, with every row present and no network access.

## Why this task exists
The real dataset is roughly a thousand buildings and the production budget is under two megabytes; a component that re-renders quadratic subtrees or copies arrays per row looks fine with three rows and stalls with two hundred. The plan sets
a deliberately generous budget (300 ms in CI) as a tripwire, not a benchmark: it catches a structural regression — a wrapper that remounts every row, a formatter called in a loop with `Intl` construction, an accidental re-sort — while
staying far from flaky. The test generates its own data because a 200-building fixture would bloat the repo and because a fixture would eventually be "fixed" instead of the component.

## Current state (preconditions)
- 06-09 done: `ResultsList` renders `buildings` in order with `BuildingRow`, the skeleton and the empty state; 06-10 added the header and the partial warning.
- `web/src/components/ResultsList.test.tsx` exists with the 06-09/06-10 cases and the render helper.
- `web/src/lib/format.ts` formatters are pure and cheap; nothing in the list constructs `Intl` per row.

## Read before you start
- `plan/stage-06-filters.md` Task 6.3 item 7 (200 buildings, budget 300 ms in CI) and the acceptance criterion "list of 200 generated buildings renders under the 300 ms budget".
- `plan/90-test-matrix.md` row W-15.
- `web/src/components/ResultsList.test.tsx` — reuse the render helper and the `Building` fixture shape; read how existing cases build props so the generated data matches the type.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/components/ResultsList.test.tsx` | the three cases below, plus one generator helper |

Nothing else may change: **no production file**. If the budget fails, the fix belongs to a separate task or a `## Result` finding — do not silently raise the budget.

## Test first (red)
1. `test_renders_two_hundred_generated_buildings_within_the_budget` — build 200 buildings with a deterministic loop (ids `b-000`…`b-199`, one to three units each, prices and sizes derived from the index), measure `performance.now()`
   immediately before and after `render`, assert the elapsed time is below `300`; a comment states this is a smoke, not a benchmark, and names the budget's source.
2. `test_all_two_hundred_rows_are_present` — `getAllByRole("listitem")` has length 200: the budget is only meaningful if nothing was dropped or virtualised away (no windowing until a task asks for it).
3. `test_generated_data_is_not_fetched` — a `fetch` spy installed with `vi.stubGlobal` is never called during the render: the test supplies data, it does not load it.

Expected red: the case does not exist yet, so the first run is a compile-level failure of the missing test helper (or `getAllByRole` returning 0 once the helper renders the wrong prop name). Run `pnpm -C web test --run ResultsList`, quote
one line.

## Implement (green)
Add one local generator (pure, deterministic, no `Math.random`, no `Date.now`) and the three cases. Keep the measurement around the `render` call only, not around data generation: the budget describes the list, not the test setup. Do not
mock timers here — `performance.now` must be the real clock.

Run `pnpm -C web test --run ResultsList` — every case green, including the new three.

## Refactor / keep it DRY
The generator lives once in this test file and returns typed `Building` values; do not duplicate the fixture shape used by the other cases. No helper in `web/src/test/` (out of scope) and no snapshot.

## Verification (how to test it)
```text
pnpm -C web test --run ResultsList
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: all `ResultsList` cases pass, including 200 rows under the budget on this machine; suite unchanged elsewhere; typecheck clean.

## Out of scope (do not do)
- Virtualisation, `React.memo`, `useMemo`, windowing, `content-visibility` hints — only add them if a separate task demands it and proves the win.
- Raising the budget, averaging runs, asserting percentiles, benchmarking `filterUnits`.
- A committed 200-building fixture or a network stub that returns a large payload.

## Definition of done
- [ ] Three cases exist, named as written, and pass.
- [ ] Red run observed and quoted; the budget constant appears once with its source in a comment.
- [ ] Suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the test file.
- [ ] `status: done`; `## Result` records the measured milliseconds.

## Commit
```text
06-11: add the large-list render budget smoke test

- points: 0.5
- red: test_renders_two_hundred_generated_buildings_within_the_budget failed (test did not exist)
- generate 200 buildings in the test and assert render under 300 ms
- assert all 200 rows render and fetch is never called
- suite: N passed
```

Commit this task alone, with the listed files only, on the trunk (`main`) — one task = one
commit, no branch, no push.

## Traps and review notes
- Measuring inside a `act`-wrapped render plus fake timers produces meaningless numbers; use the real clock and measure around `render` only.
- A budget test that passes because the list virtualised to 20 rows is worse than no test; case 2 pins row count.
- Flaky on a loaded CI box is a real risk: if it fails intermittently, record the measured spread in `## Result` and report — do not raise 300 ms without the owner.
- Generating data with `Math.random` makes failures irreproducible; derive everything from the index.
- Reviewer greps for `300` occurrences (exactly one), `Math.random`, `setTimeout`/`vi.useFakeTimers` in this file, and any production diff.

## Plan samples are untrusted
The plan states the budget and the size, not the measurement. Verify the installed Vitest/jsdom `performance.now` behaviour and the `render` timing semantics against `web/package.json` (`CLAUDE.md` §6); record deviations under
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
