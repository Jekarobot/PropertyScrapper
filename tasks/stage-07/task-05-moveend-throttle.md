---
id: 07-05
stage: 07
title: Only moveend, coalesced, loop-guarded
points: 1
owner: web
status: todo
depends_on: [07-04]
files:
  - web/src/components/MapEvents.tsx
  - web/src/components/MapView.test.tsx
---

# 07-05 · Only `moveend`, coalesced, loop-guarded

**Points:** 1 · **Stage:** 07 — Map and selection · **Owner lane:** web · **Depends on:** 07-04 · **Unblocks:** 07-06

## Goal

Make `moveend` emit at most one viewport per burst: coalesce events that arrive inside one window into a single call carrying the last bounds, and skip the call entirely when the bounds did not actually change (the programmatic-`fitBounds` feedback loop).

## Why this task exists

F8 (`plan/02-architecture.md` §5.2) says the list updates on `moveend` and—implicitly—that a drag is one update, not thirty. Leaflet fires `moveend` after every gesture, including a zoom animation and a `fitBounds` call the app itself might make; without coalescing, one drag can produce a state write per animation step and a re-render storm over the whole list. There is a second, nastier hazard: if the app ever moves the map from a bounds change, a naive `moveend` handler answers its own event forever. The guard is a comparison of the new bounds with the last emitted ones, stringified to six decimals so floating-point noise does not defeat it. This is the last map-internal task; 07-06 wires the result into the list.

## Current state (preconditions)

- 07-04 done: `web/src/components/MapEvents.tsx` subscribes `click` and `moveend`, reads `getBounds()` at event time, reports `{south, west, north, east}` and unsubscribes on unmount; the map and callbacks live in refs.
- `MapView.test.tsx` mocks `react-leaflet` and `useMap` and already has the 07-03/07-04 cases; the map click still deselects.
- 05-15 done: `setBBox` has a stable identity, so a single subscription survives parent re-renders.
- No timer, no coalescing and no bounds comparison exist yet.

## Read before you start

- `CLAUDE.md` §3 (determinism: fake timers, no real waits), §5 (one implementation of the rule), §6.
- `plan/02-architecture.md` §5.2 row F8 — 200 ms budget from `moveend` to list; §5.1 — the list follows the map.
- `plan/stage-07-map.md` Task 7.2 items 3–4 and the Traps entry "`moveend` fires on programmatic `fitBounds` too — compare with 6 decimals".
- `plan/90-test-matrix.md` row W-08 — this is the component half that the Playwright task 07-15 re-proves end to end.
- `web/src/components/MapEvents.tsx` as 07-04 left it.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/components/MapEvents.tsx` | coalescing window, bounds comparison, timer cleanup |
| modify | `web/src/components/MapView.test.tsx` | the seven cases below |

Nothing else may change: no `App.tsx`, no `useViewport`, no `lib/` edit, no map options.

## Test first (red)

Use `vi.useFakeTimers()` in every case of this group and `vi.advanceTimersByTime`. The fake map's `getBounds` returns a mutable pair of corners so a case can change the bounds between events.

1. `test_a_single_moveend_emits_once_when_the_window_closes` — one `moveend`, advance 199 ms → `onViewportChange` not called; advance 1 ms more → exactly one call with those bounds. Nothing is emitted before the window closes.
2. `test_rapid_double_moveend_emits_once_with_the_last_bounds` — bounds A at t 0, bounds B at t 100; advance 199 → no call; advance 1 → exactly **one** call, and it carries B's numbers. A burst is one update, and it is the latest viewport, not the first.
3. `test_only_moveend_is_subscribed` — the map's `on` was called exactly twice for the whole component: `"click"` and `"moveend"`. No `move`, `zoom`, `zoomstart`, `movestart` or `dragend` subscription exists to fire, and the test asserts the subscription list rather than pretending to fire them.
4. `test_identical_bounds_are_not_emitted_twice` — emit bounds A, advance past the window (one call); fire `moveend` again with bounds that stringify identically to six decimals; advance past the window → still exactly one call. This is the `fitBounds` feedback guard.
5. `test_bounds_differing_below_the_sixth_decimal_are_treated_as_equal` — a corner that moved by `1e-7` is not emitted; a corner that moved by `1e-4` is emitted. Two numbers, one comparison rule, no tolerance constant beyond the decimal count.
6. `test_unmount_clears_the_pending_timer` — fire `moveend`, unmount before the window closes, advance well past it → `onViewportChange` never called and `clearTimeout` observed (no state write into a dead tree).
7. `test_click_deselection_is_not_coalesced` — the `"click"` handler still calls `onSelect(null)` immediately, with no timer advanced; selection must never feel laggy.

Expected red: `expected 2 to be 1` for case 2 (two emissions before the throttle) and `expected 1 to be 0` for case 1 (immediate emission). Run `pnpm -C web test --run MapView`, quote one line.

## Implement (green)

- One module constant for the coalescing window — 200 ms, the same order as F8's budget, with a comment naming F8. Every event restarts the window; only the last bounds of a burst are emitted.
- `moveend` handler: read the bounds, compute a key by formatting all four numbers with six decimals and joining them; if the key equals the last emitted key, return without scheduling. Otherwise store the bounds in a ref, clear the previous timer, start a new one.
- Timer callback: re-check the key against the last emitted key (a late duplicate must not slip through), set the last-emitted key, then call the callback through the ref. Clear the pending-bounds ref afterwards.
- Cleanup on unmount clears the timer as well as unsubscribing the two listeners; the whole mechanism stays inside `MapEvents` — `App` receives plain `BBox` values and knows nothing about coalescing.
- No lodash, no debounce dependency, no `requestAnimationFrame` fallback: a single timer and a ref.

Run: `pnpm -C web test --run MapView` — seven new green, previous cases green with fake timers scoped to this file's group.

## Refactor / keep it DRY

One window constant, one key function, one place that calls `onViewportChange`. The key function is used for both the early return and the late check — do not write the four-number stringify twice. `App` (07-06) must not throttle again: two throttles in series make the real budget unmeasurable, and 07-15 measures exactly one.

## Verification (how to test it)

```text
pnpm -C web test --run MapView
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Manual check: after 07-06, `pnpm -C web dev` → one drag produces one list update after the gesture ends; a zoom produces one update; dragging back does not double-update; the network panel stays silent.

## Out of scope (do not do)

- Wiring `setBBox`/the list — 07-06.
- `fitBounds`, auto-fit, zoom-on-select, `padding`, `maxZoom` — no task in v1 needs them.
- Changing the initial center/zoom so the tests are easier to write.
- A generic `throttle`/`debounce` utility with one caller; an `AbortController` around a synchronous state write.

## Definition of done

- [ ] All seven cases exist, named as written, and pass; no case uses a real timer or `sleep`.
- [ ] Red run observed and quoted; case 2 proves one call with the last bounds, not "at most two".
- [ ] Case 4 and case 5 pin the six-decimal comparison in both directions.
- [ ] Lane suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the two files above plus this task file.
- [ ] `status: done`; `## Result` filled; one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
07-05: coalesce moveend and guard the bounds loop

- points: 1
- red: test_rapid_double_moveend_emits_once_with_the_last_bounds failed (expected 2 to be 1)
- coalesce moveend inside one 200 ms window and emit only the last bounds
- skip emission when the six-decimal bounds key equals the last emitted key
- clear the pending timer on unmount; keep map click immediate
- suite: N passed
```

Files in the commit: exactly the two listed files plus this task file.

## Traps and review notes

- Emitting on the leading edge feels faster but breaks case 2 (two calls) and makes the E2E measurement meaningless.
- Comparing raw floats without a fixed decimal count emits on every event because Leaflet's bounds jitter in the last bits; six decimals is the documented rule, not a taste.
- A guard that compares against the *pending* bounds instead of the last *emitted* ones lets an A→B→A burst emit twice.
- Leaving the timer alive across unmount produces a callback into a dead component; case 6 is the only proof.
- Coalescing `click` too makes deselection feel broken; case 7 forbids it.
- If 07-15 measures more than 200 ms from the last `moveend` to the list update, shorten the window (one constant) and record it under `## Decisions` — never widen the measurement's bound.
- Reviewers: grep `MapEvents.tsx` for `setTimeout` (one), `clearTimeout` (two: restart and cleanup) and `toFixed` with six decimals (one helper).

## Plan samples are untrusted

The plan states the six-decimal comparison in prose only. Verify `getBounds()`'s return shape and `clearTimeout`/`setTimeout` typing with the project's DOM lib (`CLAUDE.md` §6); do not copy a debounce helper from anywhere into this file. Record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
