---
id: 08-09
stage: 08
title: Mobile layout end-to-end at 375 px
points: 1
owner: web
status: todo
depends_on: [07-12, 08-07]
files:
  - web/e2e/mobile.spec.ts
---

# 08-09 · Mobile layout end-to-end at 375 px

**Points:** 1 · **Stage:** 08 — Hardening and release · **Owner lane:** web · **Depends on:** 07-12, 08-07 · **Unblocks:** 08-12

## Goal

Prove in a real browser at 375×812 that the filter bar collapses into a sheet, the map/list toggle switches panes, the page never scrolls sideways, and every tap target is at least 40 px.

## Why this task exists

Stage 07 built the mobile shell (a `Filters (N)` button opening a sheet, a list/map toggle) but only the desktop viewport is exercised by `e2e/app.spec.ts`, and jsdom cannot measure layout at all: a `min-width` on a row, a long building name or
a fixed-width control only breaks the page at a real 375 px viewport. The requirement is not "it looks fine" but two measurable facts — the document's scroll width never exceeds its client width, and a thumb can hit every control — plus the two
affordances a phone user has and a laptop does not. This is W-18 of the test matrix and the last functional evidence before the release pass (08-12). 08-07 matters here because the sheet's controls must stay reachable once the bar is hidden
behind a button.

## Current state (preconditions)

- 07-12 is done: below 768 px the filter bar collapses to a `Filters (N)` button that opens a sheet holding the five controls, and a `Map`/`List` toggle switches the bottom pane; the desktop layout is unchanged above the breakpoint.
- 07-13 is done: `web/playwright.config.ts` runs `pnpm preview` on 127.0.0.1:4173 with `testDir: "e2e"`, and a `globalSetup` copies `web/src/test/fixtures/dataset.sample.json` to `web/public/data/dataset.json`.
- 07-14/07-16 are done: `e2e/app.spec.ts` has the fifteen desktop scenarios, including the console collector and the popup check.
- 08-07 is done: one `h1`, five named landmarks, one selection status region.
- `pnpm -C web e2e` is green at the default viewport; the sample dataset is `6 units`, `3 buildings`, 2 with coordinates.

## Read before you start

- `CLAUDE.md` §3 (e2e is the only place layout may be asserted; fixtures are frozen), §5 (no `waitForTimeout`), §6, §11.
- `plan/stage-07-map.md` Task 7.4 item 5 and Task 7.5 — the mobile affordances and the Playwright setup (preview server, `globalSetup` copy, no retries).
- `plan/02-architecture.md` §5.1 — which pane holds what and the 45vh bottom map; §5.3 tokens.
- `plan/90-test-matrix.md` row W-18 and the manual acceptance line "resize to 375 px: filter sheet works, map toggles, no horizontal scroll"; `web/e2e/app.spec.ts` — helpers, selectors, console collector.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `web/e2e/mobile.spec.ts` | the five scenarios below, with a per-file mobile viewport |

Nothing else: no component, no CSS, no `playwright.config.ts`, no fixture, no unit test.

## Test first (red)

The file starts with a per-file viewport (375×812, device scale factor 2, `hasTouch: true`) so the desktop spec keeps its 1440×900 config, and uses the same `globalSetup` dataset copy. No `waitForTimeout` anywhere: every wait is an auto-retrying
locator assertion or a `poll` on an evaluated value.

1. `mobile.spec.ts::collapses the filter bar into a Filters button` — at 375 px the five controls (`Size minimum`, `Size maximum`, `Bedrooms`, `District`, `Building`) are not visible and the button named `/filters/i` is; activating it reveals
   the five controls inside the sheet; activating the same affordance again hides them, and the button's name carries the count when a filter is set.
2. `::opens the sheet and keeps every control usable` — inside the open sheet, type `60` into the size minimum and choose a district, then assert the results header count changes and the sheet is still open; close it and assert the filter is
   still applied.
3. `::switches between the list and the map` — the list is visible and the map pane is not; activating `Map` shows the map pane with a non-zero bounding box, activating `List` restores the rows; the chosen pane survives a filter change.
4. `::never scrolls horizontally` — after opening the sheet, after switching to the map and after closing it, `document.documentElement.scrollWidth` is less than or equal to `clientWidth`, and the same holds for `body`. Three states, because an
   overflowing sheet is invisible until it is open.
5. `::keeps every tap target at least 40 px` — the bounding boxes of the Filters button, the Map/List toggle, the sheet's close affordance and `Reset` are each at least 40×40 CSS px; a 39 px control fails with its measured size in the message.

Expected red: the file does not exist, so `pnpm -C web e2e` still reports 15 tests and the mobile file is not collected; on the first run the `Filters` locator fails while the sheet markup is being located. Quote the collected-test count before
and after.

Run: `pnpm -C web e2e mobile` (and `pnpm -C web e2e` for the full set).

## Implement (green)

- One spec file, `test.use({ viewport: { width: 375, height: 812 }, hasTouch: true })` at the top, a `beforeEach` that navigates to `/` and waits for the `6 units` header, and the console-error collector reused from `app.spec.ts` (or imported
  from its helper) — do not fork a second collector.
- Query by role and accessible name (`button` named `/filters/i`, the sheet by its own accessible name, the toggle by name). No CSS-class selectors, no `data-testid` added to the product for this spec, no `nth()` on structural positions.
- Assertions are `expect(locator).toBeVisible()` / `toBeHidden()` / `toHaveCount()`; for `scrollWidth` and `boundingBox` use `expect.poll` with a short timeout, because layout settles a frame after a class change.
- The horizontal-scroll check is one helper taking a state label and returning the two numbers, called at each of the three states; its failure message names the state.
- Keep the sheet's state machine as 07-12 shipped it: this spec observes, it does not redefine behaviour. If the shipped affordance has a dedicated close button instead of a toggle, assert that control and record the difference under `##
  Decisions`.
- No network beyond the preview server and no real `data/`; the spec must pass on a clean clone after `pnpm -C web e2e`'s own `globalSetup`.

Run: `pnpm -C web e2e` — 20 tests green (15 desktop + 5 mobile).

## Refactor / keep it DRY

The console collector, the navigation helper and the dataset assertion come from the existing e2e helpers, not from a copy in the new file; if `app.spec.ts` keeps them inline, extract them into the existing helper module in this commit (still
inside `e2e/`) instead of duplicating five lines. The 40 px constant and the viewport are literals in one place each.

## Verification (how to test it)

```text
pnpm -C web e2e mobile
pnpm -C web e2e
pnpm -C web test --run
```

Expected: 5 passed in the mobile spec, 12 in the full e2e run, the unit suite unchanged. Manual check: open the device toolbar at 375×812, open the sheet, scroll and confirm no horizontal scrollbar; thumb-check the Filters button and the toggle.

## Out of scope (do not do)

- Fixing layout by adding `overflow-x-hidden` to make case 4 pass: the requirement is that nothing overflows, not that the overflow is clipped. Report the offending element; the fix belongs to 07-12's component and needs its own task.
- New components, new CSS, a responsive redesign, a second breakpoint, tablets (768–1024), landscape, iOS safe areas, or visual-regression screenshots.
- A mobile Playwright project in the config, a device descriptor, retries, or `waitForTimeout`.
- Real data, real tiles (the map pane is asserted structurally), or a live site call.

## Definition of done

- [ ] All five scenarios exist, named as written, and pass at 375×812.
- [ ] Red run observed (file not collected / failing locator) and quoted in the commit body.
- [ ] `pnpm -C web e2e` runs 20 tests green with no `waitForTimeout` in the new file.
- [ ] Only `web/e2e/mobile.spec.ts` (plus an existing e2e helper if it was extracted) and this task file changed.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and the test count.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
08-09: add the 375 px mobile end-to-end spec

- points: 1
- red: pnpm -C web e2e collected 15 tests (no mobile.spec.ts)
- add the mobile spec: sheet, pane toggle, no horizontal scroll, 40 px tap targets
- suite: 20 e2e, 47 web unit tests unchanged
```

Files in the commit: `web/e2e/mobile.spec.ts`, this task file. One commit, on trunk `main`.

## Traps and review notes

- `document.body.scrollWidth` and `documentElement.scrollWidth` differ when the shell scrolls; check both, and check them while the sheet is open — a sheet wider than the viewport is the usual bug.
- `boundingBox()` returns null for an element that is not rendered; a `?? 0` fallback turns a missing control into a passed size check. Assert the box is non-null first.
- Device scale factor 2 still reports CSS pixels; compare against 40, not 80.
- Clicking `Filters` and measuring immediately races the sheet's transition — the auto-retrying assertion on the sheet's visibility is the synchronisation, not a sleep.
- Reviewers: `grep -n 'waitForTimeout' web/e2e` is empty; the scroll helper has three call sites; no new `overflow-x-hidden` in `web/src`.

## Plan samples are untrusted

The plan states the assertion as `document.documentElement.scrollWidth <= clientWidth`, names no selectors for the sheet or the toggle, and predates 07-12's markup. Read the shipped component's accessible names before writing a locator, keep the
plan's two numeric requirements, and record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |

