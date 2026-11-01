---
id: 07-14
stage: 07
title: E2E load counts and scroll isolation
points: 1
owner: web
status: todo
depends_on: [07-13]
files:
  - web/e2e/app.spec.ts
  - web/e2e/helpers/network.ts
---

# 07-14 · E2E load counts and scroll isolation

**Points:** 1 · **Stage:** 07 — Map and selection · **Owner lane:** web · **Depends on:** 07-13 · **Unblocks:** 07-15

## Goal

Add the first real Playwright spec: the app loads the frozen dataset with the documented totals, renders three rows and two bubbles, and a wheel over the map or over the list neither scrolls the page nor moves the map.

## Why this task exists

Everything before this task is a component test with mocked Leaflet — nothing has proven that the real app, the real loader, the real map and the real layout agree. This spec is the smoke test the rest of the E2E work builds on: if the totals, the row count or the bubble count are wrong here, every later scenario is measuring the wrong screen. The scroll case pins F8's other half — the wheel belongs to the page, not to the bottom map — which is the single most common "the site feels broken" report on a laptop trackpad, and one that jsdom can never show.

## Current state (preconditions)

- 07-13 done: `web/playwright.config.ts` with the preview server on 4173, the 1440×900 viewport, `testDir: "e2e"`, `retries: 0`, and a `globalSetup` that puts the sample dataset where `vite preview` serves it; `pnpm -C web e2e` exists and the config loads.
- 07-09…07-12 done: the app renders the map, the list, the panel, the no-location bucket and the legend; `pnpm -C web build` succeeds.
- 07-02 done: every bubble carries the `bubble` and `bubble-<buildingId>` classes — the only markers the spec may count.
- 05-03 done: the fixture has 6 units in 3 buildings: `p00348` (3 units, coordinates), `premier-thonglor` (2 units, coordinates), `no-location` (1 unit, `Khlong Toei`, `lat`/`lng` null).
- `web/e2e/` contains only `global-setup.ts`; `app.spec.ts` and `helpers/network.ts` do not exist.

## Read before you start

- `CLAUDE.md` §3 (no network in tests), §5 (one helper, no duplicated route stubs), §10 (no requests to the live site).
- `plan/02-architecture.md` §5.1 — the header totals, the 45 vh map band; §5.2 rows F6 and F8.
- `plan/stage-07-map.md` Task 7.5 scenarios 1–2 and the Traps entry about Playwright and tiles.
- `plan/90-test-matrix.md` rows W-06 (null coordinates), W-08 (scroll/`moveend`), W-18 (layout).
- `web/src/components/MapView.tsx` (the `bubble` classes and the initial center/zoom) and `web/e2e/global-setup.ts`.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `web/e2e/app.spec.ts` | the five cases below |
| create | `web/e2e/helpers/network.ts` | offline stubs shared by every spec file |

Nothing else may change: no component, no config, no fixture. If a selector is missing, report it — do not add test ids to production code in this task.

## Test first (red)

`helpers/network.ts` exports two helpers used from a `beforeEach`: one fulfils every tile request with a 1×1 transparent PNG (`200`, `image/png`) instead of aborting, and one fulfils `https://connex.in.th/**` with a minimal HTML document. Both keep the suite offline and — this matters for 07-16 — keep the console free of `net::ERR_FAILED` messages that a console-error collector would otherwise report.

1. `test_header_reports_the_dataset_totals` — the top bar contains `6 units` and `3 buildings` (the dataset totals, from `stats`), and the list's own header contains `2 buildings · 5 units` (only the mapped buildings are in the viewport list at load). Three different numbers on purpose: the totals must not be the viewport counts.
2. `test_three_rows_render_including_the_bucket_row` — exactly three building rows: `The Bangkok Sathorn`, `Premier Thonglor`, and one inside the `No location (1)` group (its text contains `Khlong Toei`); each name appears once, and the bucket is expanded at load.
3. `test_two_bubbles_render_and_the_unmapped_building_has_none` — `path.bubble` matches exactly 2 elements; `path.bubble-p00348` and `path.bubble-premier-thonglor` exist; `path.bubble-no-location` does not. The bubble count is read from the DOM, never from a data attribute added for the test.
4. `test_wheel_over_the_map_does_not_scroll_the_page_or_zoom` — record `window.scrollY` and the `.leaflet-map-pane` transform, send a wheel event over the map's centre, wait for the scroll position to settle with a bounded poll (no fixed sleep), then assert both unchanged: the page did not scroll and the map did not pan or zoom.
5. `test_wheel_over_the_list_does_not_scroll_the_page_or_move_the_map` — the same two assertions with the wheel over the results list. With the three-row fixture the list does not overflow, so instead of a `scrollTop` delta assert that the list container's computed `overflow-y` is `auto` (a longer list scrolls itself, not the page — its budget case is 06-11) and that `window.scrollY` and the map transform are unchanged.

Expected red: the spec cannot resolve `./helpers/network` and `path.bubble` matches 0 before the map mounts. Run `pnpm -C web e2e`, quote one failure line.

## Implement (green)

- One `beforeEach` per spec file that installs both stubs before `page.goto("/")`; no spec may rely on the real CARTO CDN. No `page.waitForTimeout`: waits are `expect(...).toHaveCount(...)`, `expect.poll` or `locator.waitFor` with an explicit timeout.
- Row locators use the accessible role and name (`getByRole("button", { name: … })`) — the rows are buttons (06-06); the bucket's toggle is a button whose name starts with `No location`.
- Bubble locators use the CSS classes 07-02 reserved (`path.bubble`, `path.bubble-<id>`); the count assertion is exact (`toHaveCount(2)`), never "greater than zero".
- For the wheel cases, dispatch the wheel at the element's bounding-box centre (`page.mouse.wheel` after `page.mouse.move`), read the map pane transform via `page.evaluate` on `.leaflet-map-pane`, and poll the scroll position until it is stable (two consecutive equal samples) before asserting. The stability wait is bounded by an explicit timeout.
- Keep the totals assertions scoped: the app header and the list header are different numbers (6/3 vs 2/5) and the case must prove both, not whichever text matches first.

Run: `pnpm -C web e2e` — five green, no retries needed (retries are 0, so a flaky assertion fails the run).

## Refactor / keep it DRY

One network helper module for all three spec tasks; a spec must not inline its own `page.route` for tiles once the helper exists. One way to locate a row, one way to locate a bubble — 07-15/07-16 import the same approach instead of inventing locators. Do not export helpers that only one case uses.

## Verification (how to test it)

```text
pnpm -C web build
pnpm -C web e2e
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: 5 passed in `pnpm -C web e2e`; the build succeeds; the unit suite is untouched; typecheck clean; `make check` green and offline. Manual check: `pnpm -C web e2e --headed` shows the map with grey (stubbed) tiles, two teal bubbles and three rows.

## Out of scope (do not do)

- Dragging, selection, the panel, filters, popups, console-error collection — 07-15/07-16.
- Screenshot comparisons, `toHaveScreenshot` baselines, video/trace recording.
- Real tiles, a tile cache, a local tile server.
- Adding `data-testid` attributes to components just for the spec.
- Retrying a failing case (`--retries`), or a `test.skip` for a flaky one.

## Definition of done

- [ ] All five cases exist, named as written, and pass with `retries: 0` in one run.
- [ ] Red run observed and quoted; the bubble count is exact and the two headers' numbers are both asserted.
- [ ] No request leaves the machine: the tile and connex.in.th routes are stubbed for every case.
- [ ] No `waitForTimeout` appears in the spec (grep it).
- [ ] Unit suite and typecheck still green; `make check` exits 0.
- [ ] `git status --short` lists only the two files above plus this task file; `status: done`; `## Result` filled; one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
07-14: prove the e2e load and scroll isolation

- points: 1
- red: test_two_bubbles_render_and_the_unmapped_building_has_none failed (expected 2, received 0)
- add the first playwright spec: totals, three rows, two bubbles
- stub tiles with a 1x1 png and connex.in.th with minimal html so the suite stays offline
- assert a wheel over the map or the list neither scrolls the page nor moves the map
- suite: N passed
```

Files in the commit: exactly the two listed files plus this task file.

## Traps and review notes

- Aborting tile requests makes Chrome log `Failed to load resource`, which 07-16's console collector would count as an error; fulfil a tiny PNG instead.
- `latitude/longitude` mistakes show up as "the map is off the coast": if two bubbles are not visible at load, check `MapView`'s initial center before touching this spec.
- Asserting `getByText("6 units")` unescaped can match the list header; the two headers carry different numbers by design.
- `waitForTimeout(500)` looks stable on a fast machine and fails on a loaded one; use bounded polls.
- Reading the bubble count from the fixture instead of the DOM hides a rendering bug; count `path.bubble`.
- Reviewers: grep `web/e2e` for `waitForTimeout` (none), `cartocdn` (only inside the helper), `test.skip` (none).

## Plan samples are untrusted

The plan's scenario list is prose. Verify the accessible names the components actually produce (rows are buttons, the bucket's toggle label) and the Leaflet DOM classes you rely on (`.leaflet-map-pane`) against the installed versions (`CLAUDE.md` §6). If a locator assumption is wrong, follow the DOM and record it under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
