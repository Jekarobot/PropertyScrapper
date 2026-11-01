---
id: 07-08
stage: 07
title: Panel units, close and maps link
points: 1
owner: web
status: todo
depends_on: [07-07, 06-07]
files:
  - web/src/components/BuildingPanel.tsx
  - web/src/components/BuildingPanel.test.tsx
---

# 07-08 · Panel units, close and maps link

**Points:** 1 · **Stage:** 07 — Map and selection · **Owner lane:** web · **Depends on:** 07-07, 06-07 · **Unblocks:** 07-09

## Goal

Fill the panel's body with the building's units rendered by the shared `UnitRow`, and add the one outbound affordance: an "Open in maps ↗" link built from the building's coordinates, replaced by a "No location" note when it has none.

## Why this task exists

The panel exists to answer "what can I rent in that tower, and where is it". The units are already rendered elsewhere by `UnitRow` (06-07) — the plan and `CLAUDE.md` §5 name that component as the single implementation of the unit line, so the panel imports it instead of copying the markup, and the safe-link behaviour (new tab, `rel="noopener noreferrer"`) comes for free. The maps link is the only way to leave the app for a real place, and it must not exist for a building whose coordinates are unknown: `lat: null` is a fact, not a formatting accident, and a link to `q=null,null` is worse than no link. Clicking that link must not close the panel — it is a navigation, not a selection.

## Current state (preconditions)

- 07-07 done: `BuildingPanel` takes two props, `building` (recomputed `Building` or `null`) and `onClose`; it renders the header (name, `district · province`, `countLabel`, `priceRange`, size range), the placeholder for `null` and the close control named `Clear selection`; its seven cases are green.
- 06-07 done: `web/src/components/UnitRow.tsx` takes `{ unit }` and renders id, beds, size, price, floor and a `view ↗` link with `target="_blank"` and `rel="noopener noreferrer"`, stopping propagation.
- 05-03 done: units carry `id`, `url`, `priceThb`, `sizeSqm`, `bedrooms`, `bathrooms`, `floor`, `updatedOn`; `p00348` has 3 units in price-then-id order.
- The whole-dataset coordinates for `p00348` are `13.7187616` / `100.5202356`; `no-location` has `lat`/`lng` `null`.

## Read before you start

- `CLAUDE.md` §3, §5 (DRY: `UnitRow` is reused; no second unit line), §6, §10 (coordinates are read as numbers only; never call a Google API).
- `plan/02-architecture.md` §5.1 — "<open on maps ↗>" in the panel block and "the unit link always opens the site in a new tab"; §5.2 row F9.
- `plan/stage-07-map.md` Task 7.3 items 3, 5 and 6.
- `plan/90-test-matrix.md` rows W-13 (links open safely) and W-06 (null coordinates).
- `web/src/components/UnitRow.tsx` — the props and the anchor behaviour the panel inherits.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/components/BuildingPanel.tsx` | the units list, the maps link, the no-location note |
| modify | `web/src/components/BuildingPanel.test.tsx` | the six cases below |

Nothing else may change: no `UnitRow` edit, no `App.tsx`, no `lib/` edit, no fixture edit.

## Test first (red)

1. `test_units_render_through_the_shared_unit_row` — with `p00348`, `vi.mock("./UnitRow")` and assert it was rendered once per unit, three times, each with the unit object from the fixture (same id). This is the DRY proof: a hand-rolled line passes a visual check and fails this case.
2. `test_units_keep_the_building_order` — the three unit ids appear in the fixture order, one each, and no id repeats.
3. `test_maps_link_uses_the_building_coordinates` — the link text is exactly `Open in maps ↗` and its `href` is exactly `https://www.google.com/maps?q=13.7187616,100.5202356` (raw dataset numbers, comma-separated, no rounding, no URL-encoding of the comma).
4. `test_maps_link_opens_a_new_tab_safely` — `target="_blank"` and `rel="noopener noreferrer"` on the same anchor (both tokens), and the anchor is not inside a button.
5. `test_building_without_coordinates_shows_no_location_instead_of_a_link` — the `no-location` building renders the text `No location`, no anchor whose `href` contains `google.com/maps`, and never `q=null` or `q=undefined`.
6. `test_clicking_the_maps_link_does_not_close_the_panel` — a click on the anchor does not call `onClose` (a surrounding click handler would otherwise treat navigation as deselection).

Expected red: `expected 0 to be 3` for the mock call count in case 1, and `Unable to find a link named Open in maps ↗` for case 3. Run `pnpm -C web test --run BuildingPanel`, quote one line.

## Implement (green)

- Units: a list of `building.units` in the given order, each rendered by `UnitRow` with the unit object. No extra wrapper that is a button, no click handler on the list, no truncation, no "showing 3 of 3".
- Empty units: render nothing but the header (a building always has ≥ 1 unit after grouping; do not add an empty-state message for this path).
- Maps link: one anchor built from `lat`/`lng` only when both are non-null, with the exact `https://www.google.com/maps?q=<lat>,<lng>` shape, `target="_blank"` and `rel="noopener noreferrer"`. Stop propagation so no surrounding handler can treat the click as a selection; do not call `onClose` and do not navigate in-app.
- No coordinates: the literal text `No location` in the same slot, muted; no tooltip, no disabled link, no placeholder URL.
- The link must not use the embedded Google key from the crawled site and must not call any Google API (`CLAUDE.md` §10): the plain `maps?q=` URL is the contract.
- One-sentence comment above the link: "opens the building's coordinates in a new tab; absent when the building has none".

Run: `pnpm -C web test --run BuildingPanel` — six new green, the 07-07 cases still green.

## Refactor / keep it DRY

`UnitRow` is imported, never re-created, and `BuildingPanel` contains no price/size/bed formatting of its own — if the panel needs a string the row already produces, it imports the formatter or the row, not a copy. The maps URL is built once, in one expression; no helper with a single caller and no URL builder shared with anything else (nothing else links to maps). The coordinate null check is one predicate for both the link and the note.

## Verification (how to test it)

```text
pnpm -C web test --run BuildingPanel
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Manual check: after 07-09, `pnpm -C web dev` → selecting `The Bangkok Sathorn` shows three unit lines with working `view ↗` links and an "Open in maps ↗" that opens Google Maps at its coordinates in a new tab; the no-location building shows "No location" and no link.

## Out of scope (do not do)

- Changing `UnitRow`, the row's link text, or its `rel` values.
- Selection sync, `App` wiring, the marker's selected style — 07-09.
- A map embed, a static-map image, a "directions" link, a phone/agent block, or any Google API call.
- Units sorting, expansion/collapse inside the panel, pagination, virtualisation.
- Pretty-printing coordinates or adding a "lat, lng" caption.

## Definition of done

- [ ] All six cases exist, named as written, and pass; the 07-07 cases still pass unedited.
- [ ] Red run observed and quoted; the `UnitRow` reuse is proven by a call-count assertion, not by eyeballing.
- [ ] The maps `href` is asserted exactly and the null-coordinate case asserts the absence of a maps anchor.
- [ ] Lane suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the two files above plus this task file.
- [ ] `status: done`; `## Result` filled; one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
07-08: render panel units and the maps link

- points: 1
- red: test_units_render_through_the_shared_unit_row failed (expected 0 to be 3)
- render building units with the shared UnitRow in fixture order
- add Open in maps with the dataset coordinates, new tab, noopener noreferrer
- show No location instead of a link when coordinates are null
- suite: N passed
```

Files in the commit: exactly the two listed files plus this task file.

## Traps and review notes

- Copying the unit line into the panel is the exact duplication `CLAUDE.md` §5 forbids; case 1 makes the copy fail.
- `q=null,null` is a real URL and loads a random place; case 5 forbids it rather than checking the text loosely.
- A link without `noopener` hands the third-party page a window reference (W-13); the exact `rel` string is asserted.
- Wrapping the link in a clickable container re-introduces the "navigation selects the building" bug 06-07 already fixed once.
- Using the site's embedded Google key is a licensing and privacy violation (`CLAUDE.md` §10); reviewers grep for `key=` and `maps.googleapis`.
- Reviewers: grep `BuildingPanel.tsx` for `view ↗` (none — it comes from `UnitRow`) and `google.com/maps` (one).

## Plan samples are untrusted

The plan's link shape is prose (`https://www.google.com/maps?q=<lat>,<lng>`). Verify `UnitRow`'s actual props and the `Building`/`Unit` field names in the repo before using them (`CLAUDE.md` §6); confirm the exact coordinate strings in the fixture instead of reformatting them. Record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
