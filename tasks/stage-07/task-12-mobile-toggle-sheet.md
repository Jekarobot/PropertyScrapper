---
id: 07-12
stage: 07
title: Mobile map toggle and filter sheet
points: 2
owner: web
status: todo
depends_on: [07-09, 06-04]
files:
  - web/src/state/useIsNarrow.ts
  - web/src/state/useIsNarrow.test.ts
  - web/src/components/FilterSheet.tsx
  - web/src/components/FilterSheet.test.tsx
  - web/src/test/media.ts
  - web/src/test/setup.ts
  - web/src/App.tsx
  - web/src/App.test.tsx
---

# 07-12 · Mobile map toggle and filter sheet

**Points:** 2 · **Stage:** 07 — Map and selection · **Owner lane:** web · **Depends on:** 07-09, 06-04 · **Unblocks:** 08-09, 08-10

## Goal

Below 768 px show one of the list or the map with a `Map` toggle, and collapse the filter bar into a single `Filters (N)` button that opens the same filter bar in a modal sheet closable with Escape and a basic focus trap.

## Why this task exists

At 375–767 px a 340 px panel, a filter bar and a 45 vh map cannot coexist: the layout would either overflow horizontally (W-18's failure) or shrink everything to unreadable. The product is used on a phone to answer "what is near me right now", so the list and the map take turns and the filters move behind one button. The sheet must reuse `FilterBar` rather than re-implementing five controls — a second copy would drift in debounce, validation and labels. This task is also the reason W-18 has a component-level half at all: jsdom cannot measure overflow, so the *behaviour* (which region is visible, what is focusable) is proven here and the pixel proof is 08-09's Playwright job.

## Current state (preconditions)

- 07-09 done: `App` owns one selection and mounts `MapView`, `ResultsList`, `BuildingPanel` and the no-location bucket; `App.test.tsx` mocks `MapView` and stubs `fetch`.
- 06-04 done: `FilterBar` has the named group `Search filters`, the Reset button, the polite live count and the documented Tab order; it takes `filters`, `resultCount`, `onChange`, `onReset`. It holds no state of its own.
- 07-10 done by lane order: the legend and the bucket are mounted.
- 07-01 done: `web/src/test/setup.ts` has a `matchMedia` stub whose default answer is desktop.
- No `useIsNarrow`, no `FilterSheet`, no mobile chrome exist.

## Read before you start

- `CLAUDE.md` §3 (fake timers/events, no skipped tests), §5 (reuse `FilterBar`; no new dependency for a dialog), §6.
- `plan/02-architecture.md` §5.2 row F10 (state in React only) and §5.1 (the filter bar row, the map band); `plan/stage-07-map.md` Task 7.4 item 5.
- `plan/90-test-matrix.md` row W-18 — its unit half is this task, its E2E half is 08-09.
- `web/src/components/FilterBar.tsx` and its props — the sheet is a container around it, not a rewrite.
- The installed `@testing-library/user-event` for Tab/Shift+Tab and Escape.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `web/src/state/useIsNarrow.ts` | one media-query hook, `(min-width: 768px)` inverted |
| create | `web/src/state/useIsNarrow.test.ts` | the three cases below |
| create | `web/src/components/FilterSheet.tsx` | the `Filters (N)` button, the modal sheet, focus handling |
| create | `web/src/components/FilterSheet.test.tsx` | the three cases below |
| create | `web/src/test/media.ts` | `setViewportWidth(px)` driving the single `matchMedia` fake |
| modify | `web/src/test/setup.ts` | install that fake instead of the fixed 07-01 stub |
| modify | `web/src/App.tsx` | the narrow layout, the toggle, the sheet button |
| modify | `web/src/App.test.tsx` | the two cases below |

Nothing else may change: no `FilterBar`, `MapView`, `ResultsList` or `BuildingPanel` internals, no `lib/` edit, no CSS framework change.

## Test first (red)

`web/src/test/media.ts` exports `setViewportWidth(px)`; `setup.ts` installs one `matchMedia` whose `matches` is computed from the current width and which dispatches a `change` event to registered listeners when the width crosses the query. Tests set the width, never a per-file fake.

1. `useIsNarrow.test.ts::test_returns_whether_the_viewport_is_below_768` — at 1280 → `false`; at 767 → `true`; at 768 → `false` (the breakpoint is exclusive at the bottom, matching Tailwind's `md`).
2. `useIsNarrow.test.ts::test_reacts_to_a_width_change` — start narrow, render, `setViewportWidth(1024)` inside `act`: the hook returns `false` on the next render without a remount.
3. `useIsNarrow.test.ts::test_removes_the_listener_on_unmount` — after unmount, changing the width does not call the state setter (no "update on unmounted component" warning, listener count back to zero).
4. `FilterSheet.test.tsx::test_button_shows_the_active_filter_count_and_opens_a_named_dialog` — with two non-default filters the trigger reads `Filters (2)`; pressing it opens a `role="dialog"` whose accessible name is `Filters` containing the real `FilterBar` (its `Search filters` group is present once).
5. `FilterSheet.test.tsx::test_escape_closes_the_sheet_and_restores_focus` — Escape closes the dialog, the trigger is `document.activeElement`, and `FilterBar`'s live count is no longer in the tree.
6. `FilterSheet.test.tsx::test_tab_is_trapped_inside_the_sheet` — from the last focusable element Tab moves to the first inside the dialog, and Shift+Tab from the first moves to the last; focus never lands on the page behind while the sheet is open.
7. `App.test.tsx::test_narrow_viewport_shows_one_region_at_a_time_and_the_map_toggle_switches_it` — at 375 px the list is visible and the map region is not; the `Map` button has `aria-pressed="false"`, pressing it shows the map and hides the list with `aria-pressed="true"`; pressing again returns to the list.
8. `App.test.tsx::test_mobile_filter_chrome_appears_only_when_narrow` — at 375 px the filter bar is not rendered, the `Filters (1)` button is, and opening the sheet shows the same five controls; at 1280 px the bar is rendered and neither the `Filters (…)` button nor the `Map` toggle exists.

Expected red: `Failed to resolve import "./useIsNarrow"` and `Unable to find role="button" and name "Filters (1)"`. Run `pnpm -C web test --run useIsNarrow` and `--run FilterSheet`, quote one line each.

## Implement (green)

- `useIsNarrow()` → `boolean`, reading `(min-width: 768px)` and returning its negation; subscribes with `addEventListener("change", …)` and unsubscribes on unmount; one docstring: "true below 768 px". No global state, no context, no resize listener of its own.
- `web/src/test/media.ts`: one module-level width plus a single `matchMedia` implementation exported for `setup.ts` to install, and `setViewportWidth` for tests. `setup.ts` imports it; delete the fixed 07-01 body rather than keeping both.
- `FilterSheet` props `{ filters, resultCount, onChange, onReset }` (whatever `FilterBar` needs) plus nothing else. It renders the `Filters (N)` trigger — N is the count of filter values that differ from the defaults, computed once from the same defaults `FilterBar` uses — and, when open, a dialog with the accessible name `Filters` containing `FilterBar`. `onClose` semantics: Escape, a labelled close button (`Close filters`), and a click on the backdrop. Focus moves into the dialog on open, returns to the trigger on close, and Tab/Shift+Tab wrap. No portal library, no focus-trap dependency: a keydown handler over the dialog's focusable descendants.
- `App`: when narrow, render either the list region or the map region (never both), with a `Map` button (`aria-pressed`) toggling them; render `FilterSheet` in place of the desktop `FilterBar`. When wide, the current layout is unchanged. Keep one `FilterBar` implementation — the sheet and the desktop bar must be the same component with the same props.
- No fixed pixel widths in the new markup and no `overflow-x-hidden` masking: the narrow layout uses the same fluid containers, and the pixel proof belongs to 08-09.

Run both target suites, then `pnpm -C web test --run` — all green.

## Refactor / keep it DRY

One breakpoint constant (768), one `matchMedia` fake (in `media.ts`), one `FilterBar`, one filter-defaults comparator (shared with `FilterBar`'s Reset predicate — import it, do not copy it). The `Filters (N)` count and the Reset disabled state must agree by construction. `App` must not grow a second copy of the filter props for the sheet.

## Verification (how to test it)

```text
pnpm -C web test --run useIsNarrow
pnpm -C web test --run FilterSheet
pnpm -C web test --run
pnpm -C web typecheck
make check
```

(`pnpm -C web e2e` exists from 07-13; 08-09 adds the mobile viewport case that measures `scrollWidth`.) Manual check: resize `pnpm -C web dev` below 768 px → one region, a `Map` toggle, a `Filters (N)` button; open the sheet, Tab around it, press Escape; above 768 px → the normal two-column layout with the bar; no horizontal scrollbar in either width.

## Out of scope (do not do)

- A design-system dialog, a focus-trap dependency, `inert`, `aria-modal` polyfills beyond one attribute; hiding overflow with `overflow-x-hidden` instead of fixing widths (08-09 measures the real thing).
- URL sync of the open sheet, animated transitions, swipe gestures, reduced-motion handling (08-10); any edit to `FilterBar`'s controls, debounce or validation, or a second filter implementation for mobile.

## Definition of done

- [ ] All eight cases exist, named as written, and pass; earlier `App` and `FilterBar` cases still pass unedited.
- [ ] Red run observed and quoted for both target suites.
- [ ] One `matchMedia` fake and one breakpoint constant; the 07-01 stub body is gone, not duplicated.
- [ ] Escape closes the sheet and focus returns to the trigger; Tab and Shift+Tab wrap.
- [ ] Lane suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the eight files above plus this task file.
- [ ] `status: done`; `## Result` filled; one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
07-12: add the mobile map toggle and filter sheet

- points: 2
- red: test_mobile_filter_chrome_appears_only_when_narrow failed (Unable to find role="button" and name "Filters (1)")
- add useIsNarrow on a single drivable matchMedia fake
- add FilterSheet wrapping the existing FilterBar with Escape and a Tab trap
- show one region at a time below 768 px with an aria-pressed Map toggle
- suite: N passed
```

Files in the commit: exactly the eight listed files plus this task file.

## Traps and review notes

- jsdom cannot measure overflow: do not assert `scrollWidth` here (it is always 0) and do not claim W-18 is proven; the Playwright case in 08-09 owns it.
- A per-test `window.matchMedia = vi.fn()` in every file duplicates the fake and hides the shared breakpoint; `setViewportWidth` exists for this.
- Rendering both regions and hiding one with CSS passes a visual check and keeps the map's `MapContainer` mounted — that is intended for the map's state, but the hidden region must not stay focusable; use `hidden`/`display:none`, not `opacity-0`.
- A copied `FilterBar` for mobile would drift on the Reset predicate and the live count; the sheet must render the real component.
- Escape must stop at the sheet: an Escape handler on the page would also clear the selection (07-12 must not add one).
- Reviewers: grep `web/src` for `matchMedia` (one fake in `media.ts`), `768` (one constant) and `Filters (` (one component).

## Plan samples are untrusted

The plan's mobile item is one sentence. The breakpoint, the `Filters (N)` count semantics and the focus rules are decisions this task must make; verify `FilterBar`'s props and the existing pluralisation/`aria-live` behaviour in the repo (`CLAUDE.md` §6) and record the decisions under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
