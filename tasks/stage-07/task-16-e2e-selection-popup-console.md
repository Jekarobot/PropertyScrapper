---
id: 07-16
stage: 07
title: E2E selection, filters, popup and console
points: 2
owner: web
status: todo
depends_on: [07-15]
files:
  - web/e2e/app.spec.ts
---

# 07-16 · E2E selection, filters, popup and console

**Points:** 2 · **Stage:** 07 — Map and selection · **Owner lane:** web · **Depends on:** 07-15 · **Unblocks:** 08-09

## Goal

Close the E2E loop: a bubble click fills the panel with that building and highlights its row, the `beds` filter changes the bubble count and Reset restores it, a unit link opens the real listing URL in a new tab, and the whole flow produces no console error.

## Why this task exists

W-09, W-13 and W-19 are the last rows of the stage that no test has proven in a browser: the component tests assert mock props, and only a real page can show that clicking a bubble, reading the panel, filtering the bubbles and opening a popup all coexist without a console error or a lost selection. The console collector is the cheapest regression net in the project — it catches the "Cannot read properties of null" that a mocked Leaflet can never produce, and it is the row the release task 08-12 leans on. The popup case also enforces the one rule about the old site that must never break: the app links out, it never loads `connex.in.th` itself.

## Current state (preconditions)

- 07-15 done: `web/e2e/app.spec.ts` has the load, scroll and drag cases; `web/e2e/helpers/map.ts` exposes the drag helper and the shared readers; `pnpm -C web e2e` passes with `retries: 0`.
- 07-14 done: `web/e2e/helpers/network.ts` fulfils tile requests with a 1×1 PNG and `https://connex.in.th/**` with a minimal HTML document — offline, and free of resource-load console errors.
- 07-07/07-08 done: the panel shows the building's name, its units (`UnitRow`, each with a `view ↗` link to `unit.url`) and the units' order.
- 06-01/06-02 done: the filter bar's Bedrooms control has the accessible names the browser test needs; 06-04 done: a `Reset` button exists and is enabled only when a filter is non-default.
- 07-02 done: bubbles carry `bubble-<buildingId>`; 06-06 done: the selected row carries `aria-current="true"`.

## Read before you start

- `CLAUDE.md` §3 (no skips, no weakened assertions), §10 (never fetch or store a live detail page; no Google API).
- `plan/02-architecture.md` §5.2 rows F2 (`beds` exact, `4+` = ≥ 4), F9 (selection), F11; §5.1 — the unit link always opens the site in a new tab.
- `plan/stage-07-map.md` Task 7.5 scenarios 4–7.
- `plan/90-test-matrix.md` rows W-09, W-13, W-19.
- `web/src/test/fixtures/dataset.sample.json` — which units have 2 bedrooms (the expected bubble count after `beds=2` must be derived from it, not guessed).
- `web/src/components/BuildingPanel.tsx` and `web/src/components/FilterBar.tsx` — the accessible names the locators rely on.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/e2e/app.spec.ts` | the five cases below and the console collector |

Nothing else may change: no helper rewrites (extend `helpers/` only if a reader is genuinely shared by two cases and say so), no component, no config, no fixture.

## Test first (red)

Install the console collector in the spec's setup: push every `console` message of type `error` and every `pageerror` into an array, with its text; never filter by message content (the network stubs exist so that no resource error occurs in the first place).

1. `test_clicking_a_bubble_fills_the_panel_with_the_building_and_its_units` — click `path.bubble-p00348`: the left panel's heading is `The Bangkok Sathorn`, it shows `3 units` and the three unit ids from the fixture, and the buttons `Clear selection` and the `Open in maps ↗` link exist.
2. `test_the_selected_building_row_is_highlighted` — after the same click, exactly one row carries `aria-current="true"` and it is the `The Bangkok Sathorn` row; the other two rows do not.
3. `test_beds_filter_reduces_the_bubbles_and_reset_restores_them` — record the bubble count (2 at load); choose `2` in the Bedrooms control; assert the bubble count equals the number of buildings with a 2-bedroom unit in the fixture (read the fixture and assert the exact number, 0 included — if the fixture has none, the list goes to its empty state and the bubble count is 0, which is also recorded in `## Result`); press `Reset` and assert the bubble count is 2 again and the row set matches the recorded one.
4. `test_unit_link_opens_the_listing_in_a_new_tab` — with the panel open, `page.waitForEvent("popup")` around the click on the first unit's `view ↗` link; the popup's URL starts with `https://connex.in.th/en/detail/` (the stub answers it, so nothing leaves the machine). Assert the original page's URL did not change.
5. `test_no_console_errors_during_the_main_flow` — run the whole scenario in one test: load, click a bubble, read the panel, set `beds=2`, reset, open the unit popup, close it; then assert the collected errors array is empty. One error is a failure, and the message is printed as the assertion's diff.

Expected red: `Unable to find role="heading" and name "The Bangkok Sathorn"` after the bubble click (the panel is not filled) and `expected [ 'Failed to load resource…' ] to deeply equal []` for the console case if the tile stub is missing. Run `pnpm -C web e2e`, quote one line.

## Implement (green)

- Bubble click: locate by `path.bubble-<id>` and click its centre; Leaflet's marker path is the interactive element, so no force-click and no `dispatchEvent` shortcut — a real click proves the handler is wired.
- Panel assertions use roles and exact strings: heading level for the name, text for the count, link for the maps URL, button for `Clear selection`. No CSS-class assertions on the panel.
- Filters: use the Bedrooms control by its accessible name (the `<select>`/fieldset labels from 06-02), assert the bubble count and, after `Reset`, both the count and the row set. The expected count comes from the fixture, computed in the test, not from a hard-coded guess.
- Popup: `Promise.all([page.waitForEvent("popup"), link.click()])`, then assert the popup URL prefix and that the source page's URL is unchanged; close the popup so the console collector covers its lifetime too.
- Console: one collector per test file installed before `goto`, assert empty at the end of the flow case; do not add a second collector inside a case.
- No `waitForTimeout`; every wait is a locator expectation or a bounded poll.

Run: `pnpm -C web e2e` — five new green, the 07-14/07-15 cases still green, one run, no retries.

## Refactor / keep it DRY

One way to click a bubble, one way to read the row set, one console collector. The expected 2-bedroom count is derived from the fixture in one helper, not repeated per case. Do not add a second network stub or a per-case route override.

## Verification (how to test it)

```text
pnpm -C web build
pnpm -C web e2e
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: 15 passed in `pnpm -C web e2e` (5 from 07-14, 5 from 07-15, 5 here), suite green with `retries: 0`, `make check` green. Manual check: `pnpm -C web e2e --headed` shows the bubble click filling the panel, the beds filter thinning the bubbles, Reset restoring them, and a new tab opening on the stubbed listing.

## Out of scope (do not do)

- A live request to `connex.in.th` or Google Maps (`CLAUDE.md` §10); the stubs stay. No backend either: the app is a static file and the suite is offline.
- Mobile viewport cases (08-09), reduced motion (08-10), screenshot baselines.
- Changing components to satisfy a locator; if a control has no accessible name, report it rather than adding a test id.
- Retries, `test.skip`, or filtering the console collector by message text.

## Definition of done

- [ ] All five cases exist, named as written, and pass in a single `retries: 0` run.
- [ ] Red run observed and quoted; the console case asserts an empty array, not "no errors of a kind".
- [ ] The `beds=2` expected bubble count is derived from the fixture and recorded in `## Result`.
- [ ] The popup assertion checks the URL prefix and that the source page did not navigate.
- [ ] Unit suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the spec file plus this task file; `status: done`; `## Result` filled; one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
07-16: prove selection, filters, popup and a clean console

- points: 2
- red: test_clicking_a_bubble_fills_the_panel_with_the_building_and_its_units failed (Unable to find role="heading" and name "The Bangkok Sathorn")
- click a bubble, assert the panel, the units and the aria-current row
- filter beds=2 and restore with Reset, deriving the expected bubble count from the fixture
- open a unit link in a popup and assert the connex.in.th detail URL prefix
- collect console errors and page errors for the whole flow and require none
- suite: N passed
```

Files in the commit: exactly the spec file plus this task file.

## Traps and review notes

- Filtering the console collector to ignore "Failed to load resource" hides the very regressions it exists for; stub the network instead.
- A `force: true` click on a bubble bypasses Leaflet's hit testing and can pass while the real marker is unclickable.
- A popup that is not awaited races the assertion; `waitForEvent("popup")` must be armed before the click, and the popup must be closed to keep the run tidy.
- Asserting "fewer bubbles than before" passes when the filter breaks the app to zero; derive the exact expected count from the fixture.
- `beds=2` may legitimately match nothing in the sample; that is a finding to record, never a reason to edit the fixture (`CLAUDE.md` §3: fixtures are frozen).
- A second console collector inside the same case duplicates messages and makes the failure unreadable.
- Reviewers: grep `web/e2e` for `force: true` (none), `waitForTimeout` (none), `console` handlers (one), `test.skip` (none).

## Plan samples are untrusted

The plan's scenario list is prose; the fixture determines the actual 2-bedroom count and the actual unit URLs. Verify the URLs in `dataset.sample.json` and the components' accessible names in the repo before writing locators (`CLAUDE.md` §6). If the fixture's unit URLs do not start with `https://connex.in.th/en/detail/`, fail and report — do not edit the fixture.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
