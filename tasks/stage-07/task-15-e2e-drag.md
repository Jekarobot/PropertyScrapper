---
id: 07-15
stage: 07
title: Investigate and prove drag updates the list
points: 3
owner: web
status: todo
depends_on: [07-14]
files:
  - web/e2e/app.spec.ts
  - web/e2e/helpers/map.ts
---

# 07-15 · Investigate and prove drag updates the list

**Points:** 3 · **Stage:** 07 — Map and selection · **Owner lane:** web · **Depends on:** 07-14 · **Unblocks:** 07-16

## Goal

Establish how to drag the Leaflet map reliably in Playwright, then prove that a drag changes the right-hand row set, that the URL and hash do not change, and that the list settles inside the F8 budget — with the findings written into `## Result`.

## Why this task exists

This is the task the whole stage exists for. `moveend` → bounds → `setBBox` → list is invisible to jsdom: Leaflet's drag needs layout, real pointer events and a real animation loop, so every earlier claim about it is a component test with a mocked map. A working drag test also has non-obvious mechanics — Leaflet listens on the document after `mousedown`, needs several intermediate `mousemove`s to pass its drag threshold, and ends the gesture on `mouseup` with inertia — and one honest hazard: a naive test passes on a fast machine and fails in CI. The investigation is therefore part of the deliverable, and it must land in `## Result` before the assertions are trusted.

## Current state (preconditions)

- 07-14 done: `web/e2e/app.spec.ts` has the load, rows, bubbles and scroll cases; `web/e2e/helpers/network.ts` stubs tiles and `connex.in.th`; `pnpm -C web e2e` passes with `retries: 0`.
- 07-06/07-05 done: the map reports coalesced `moveend` bounds, `App` cuts the list with them.
- 07-02 done: bubbles carry `bubble` and `bubble-<id>`; 07-10 done: the `No location (N)` bucket renders under the list.
- The map's initial center/zoom are `MapView`'s constants (`[13.7563, 100.5018]`, 12), not test-controlled.

## Read before you start

- `CLAUDE.md` §3 (no arbitrary waits, no skipped tests), §4 (a 3-point task writes down what it learned before coding), §6.
- `plan/02-architecture.md` §5.2 row F8 — `moveend` updates the list within 200 ms; §5.1 — the list shows the viewport's buildings.
- `plan/stage-07-map.md` Task 7.5 scenario 3 and the acceptance line "dragging the map updates the list within 200 ms".
- `plan/90-test-matrix.md` row W-08 — its E2E half is this file.
- `web/src/components/MapEvents.tsx` (the coalescing window) and `web/src/components/MapView.tsx` (the initial center).

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/e2e/app.spec.ts` | the five cases below |
| create | `web/e2e/helpers/map.ts` | one drag helper, the map-pane/row readers |

Nothing else may change: no component, no config, no fixture, no tolerance change. A finding that needs a product change goes to `## Result` and to the owner.

## Test first (red)

Start with the investigation, not the assertions: a scratch run (kept out of the commit) that answers — where is the map's draggable pane, which mouse sequence Leaflet accepts, how many `moveend` events one gesture produces, how long after `mouseup` the list changes, and how many 400 px strokes it takes for the row set to change at zoom 12. Record the answers in `## Result` **before** writing the final cases, as `CLAUDE.md` §4 requires.

1. `test_dragging_the_map_west_changes_the_visible_rows` — record the row names, drag the map west by 400 px strokes from the centre of the map's draggable pane (not the legend, not the zoom control), bounded at three strokes, and assert the row set is different from the recorded one. A stroke count of one is allowed only if the investigation measured that it changes the set; record the count actually used.
2. `test_drag_does_not_change_the_url_or_hash` — `page.url()` is byte-identical before and after the drag (no hash, no query, no history entry). F10: nothing about the viewport is URL state.
3. `test_the_list_settles_after_the_drag_and_stops_changing` — after the last stroke, poll the row set until two consecutive samples are equal (bounded timeout), then assert it stays equal for one more sample: no post-gesture flicker, and the wait is a predicate, never a fixed sleep.
4. `test_list_updates_within_the_200ms_budget_after_the_last_moveend` — install a `MutationObserver` on the results list and record `performance.now()` at the synthetic `mouseup` and at the first mutation after it, in one `page.evaluate`; assert the delta is within 200 ms. If the measurement cannot resolve the interval (inertia plus frame granularity), record the measured number in `## Result` and the assertion's real ceiling — do not silently widen it.
5. `test_a_second_drag_keeps_updating_without_a_reload` — drag again after the first update and assert the rows change again and the page did not navigate (same `page.url()`, no full document load): the `moveend` subscription is not one-shot.

Expected red: `expected [ … ] not to equal [ … ]` (the row set is unchanged because the drag never reached the map) or a timeout waiting for the row set to change. Run `pnpm -C web e2e`, quote one line.

## Implement (green)

- `helpers/map.ts` exports one `dragMap(page, dx, dy)` and the readers the cases share: the map pane's box, the current row names, the settle predicate. Coordinates are computed from the live box, never hard-coded pixel constants that assume the 1440×900 viewport (use the box, assert the viewport once).
- The drag: move to the pane's centre, `mouse.down()`, 8–12 `mouse.move` steps with a small delay proportional to the distance (Leaflet ignores a single jump in some browsers), then `mouse.up()`. Do not press on the legend, the zoom control or a bubble: a press on a bubble selects it instead of dragging.
- Every wait is a predicate: `expect.poll`, `locator.waitFor`, or `page.waitForFunction` with an explicit timeout. No `waitForTimeout` anywhere.
- Bubble/row locators come from 07-14's approach; do not add a second way to find a row.
- If the investigation shows that one 400 px stroke cannot change the row set at zoom 12 (the two mapped Bangkok buildings sit inside a ~53 km-wide view), keep the stroke size and repeat the gesture — never change `MapView`'s initial center or zoom to make the test pass, and never assert "changed" without recording what changed.

Run: `pnpm -C web e2e` — five new green, the 07-14 cases still green, one run, no retries.

## Refactor / keep it DRY

One drag helper, one row reader, one settle predicate, shared by this task and 07-16. The stroke size and step count are named constants in the helper with a comment pointing at `## Result`'s measurements. No spec-local copy of the drag sequence.

## Verification (how to test it)

```text
pnpm -C web build
pnpm -C web e2e
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: 10 passed in `pnpm -C web e2e` (5 from 07-14, 5 here); the suite green with `retries: 0`; `make check` green. Manual check: `pnpm -C web e2e --headed` shows the drag moving the map and the row list changing once, with no URL change.

## Out of scope (do not do)

- Selection, the panel, filters, popups, console collection — 07-16.
- Changing the map's centre, zoom, inertia or animation options to make dragging easier.
- Retying or spacing the test with sleeps; `test.slow()` to hide a timing problem.
- Asserting a specific row *set* derived from coordinates by hand: the assertion is "changed and stable", with the observed sets in `## Result`.

## Definition of done

- [ ] The investigation findings are written in `## Result` **before** the assertions are claimed as green: the drag sequence, the `moveend` count per gesture, the measured `mouseup`→list delta, the strokes needed.
- [ ] All five cases exist, named as written, and pass in a single `retries: 0` run.
- [ ] Red run observed and quoted; no case uses `waitForTimeout`.
- [ ] The URL/hash assertion compares the full URL string, not a fragment.
- [ ] Unit suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the two files above plus this task file; `status: done`; one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
07-15: prove that dragging the map updates the list

- points: 3
- red: test_dragging_the_map_west_changes_the_visible_rows failed (row set unchanged after the drag)
- record the drag mechanics: pane centre, stepped mouse moves, mouseup then a bounded settle poll
- assert the row set changes, the url and hash do not, and the list settles after the gesture
- measure mouseup to first list mutation against the 200 ms budget
- suite: N passed
```

Files in the commit: exactly the two files above plus this task file.

## Traps and review notes

- Pressing on the legend, the zoom control or a bubble: the first two swallow the drag, the third selects a building — the press point comes from the pane's box, not the map's outer container.
- A single `mouse.move` jump can be dropped by Leaflet's drag detection; the stepped move exists for this and is measured, not guessed.
- `waitForTimeout` "fixes" a flaky test by making it slower and still flaky; the settle predicate is the contract.
- Leaflet's drag inertia keeps the map moving for a moment after `mouseup`, so the last `moveend` (and the coalesced emit) arrives after the release; the budget measurement must start at the last `moveend`, and 07-05's window is part of the measured interval.
- If the measured delta exceeds 200 ms because of that window, the fix is to shorten the window in `MapEvents` (recorded under `## Decisions`), not to soften the assertion or add `test.slow()`.
- A URL assertion that only checks for the absence of `#` misses a `history.replaceState`; compare the whole URL.
- Reviewers: grep `web/e2e` for `waitForTimeout` (none), `mouse.down` (one helper), and for a changed `13.7563` in `MapView.tsx` (unchanged).

## Plan samples are untrusted

The plan says "drag the map by (−400, 0)" without a mechanism or a measurement method. Leaflet's drag behaviour, inertia and event ordering must be verified in the installed `leaflet@1.9.4` source and against the real browser (`CLAUDE.md` §6) — that verification *is* this task's investigation. Record every finding under `## Result` and any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
| Drag mechanics (pane point, steps, mouseup) | |
| `moveend` events per gesture | |
| `mouseup` → first list mutation | |
| Strokes needed to change the row set | |
