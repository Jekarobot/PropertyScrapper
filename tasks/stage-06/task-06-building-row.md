---
id: 06-06
stage: 06
title: Building row with selection and long-name truncation
points: 1
owner: web
status: todo
depends_on: [05-10]
files:
  - web/src/components/BuildingRow.tsx
  - web/src/components/BuildingRow.test.tsx
---

# 06-06 · Building row with selection and long-name truncation
**Points:** 1 · **Stage:** 06 — Filters and results list · **Owner lane:** web · **Depends on:** 05-10 · **Unblocks:** 06-08, 06-09

## Goal
Create `BuildingRow`: one keyboard-reachable button per building showing name, district, unit count, price range and maximum size, with selection styling and single-line name truncation.

## Why this task exists
The right-hand list groups units by building, so each row must summarise a building in one glanceable line and be selectable, because selection is what fills the left panel and highlights the bubble in stage 07 (`plan/02-architecture.md`
§5.1). The row is the only place a visitor picks a building, so it must be a real button — not a div with a click handler — and it must show the **recomputed** ranges of whatever survived filtering, or a filtered row advertises prices of
hidden units (W-12). `ResultsList` (06-09) and the expanded row (06-08) both build on this component.

## Current state (preconditions)
- 05-10 done: `lib/group.ts` exports `groupByBuilding`, `countLabel(n)` (`1 unit` / `3 units`) and `priceRange(building)` (`45–50k`); 05-12 exports `sqm`.
- 05-03 done: `Building` and `Unit` types include `id`, `name`, `district`, `province`, `unitCount`, `priceMin`, `priceMax`, `sizeMin`, `sizeMax`, `units`, `lat`, `lng`.
- `dataset.sample.json` has 3 buildings (6 units): `The Bangkok Sathorn` (3 units, `Sa Thon`), `Premier Thonglor` (2 units, `Watthana`), one no-location building (1 unit, `Khlong Toei`).
- `web/src/components/BuildingRow.tsx` and its test do **not** exist.

## Read before you start
- `plan/stage-06-filters.md` Task 6.3 — the row anatomy block and items 1, 2, 6.
- `plan/02-architecture.md` §5.1 (row shows name, district, unit count, price range; listed by name then id), §5.3 (accent `#0f766e`, `font-medium` names, `tabular-nums`, 10 px radius, `shadow-sm` only where specified).
- `plan/90-test-matrix.md` rows W-12 (recomputed ranges) and W-09 (selection).
- `web/src/lib/group.ts` and `web/src/lib/format.ts` — the three formatters must be reused, not redone.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `web/src/components/BuildingRow.tsx` | the row component and its props type |
| create | `web/src/components/BuildingRow.test.tsx` | the seven cases below |

Nothing else may change: no `ResultsList` yet, no `App.tsx`, no `lib/` edits.

## Test first (red)
1. `test_row_shows_name_district_count_and_ranges` — render `The Bangkok Sathorn` (3 units): the row contains the name, `Sa Thon`, `3 units`, the exact `priceRange` string for its prices and the maximum size via `sqm`; numbers are literals
   copied from the fixture.
2. `test_row_is_a_button_named_after_the_building` — `getByRole("button", { name: /The Bangkok Sathorn/ })` resolves, is a native `button`, and the row contains no link yet (unit links arrive in 06-08, which adds a second control; do not
   assert that this is the only control).
3. `test_clicking_the_row_selects_the_building` — a click calls `onSelect` exactly once with the building's `id` (`p00348` for the sample dataset).
4. `test_selected_row_has_aria_current_and_an_accent_border` — `selected` true sets `aria-current="true"` on the button and the accent border class (assert the class token, per the plan).
5. `test_unselected_row_has_no_aria_current` — `selected` false carries no `aria-current` attribute at all (not `"false"`).
6. `test_long_name_truncates_on_one_line_with_a_title_attribute` — a 60-character name renders as one line: the name element carries the full name in `title`, a `truncate`-style class, and no class that allows wrapping (assert the token, not
   computed CSS).
7. `test_keyboard_activation_selects_the_row` — focus the button and press Enter and Space: each calls `onSelect` with the id (native button behaviour, pinned against a future div-plus-keydown regression).

Expected red: `Failed to resolve import "./BuildingRow"`. Run `pnpm -C web test --run BuildingRow`, quote one line.

## Implement (green)
Export `BuildingRow` with props `{ building, selected, onSelect }`:

- `building` is the **already recomputed** object for the current filter set; the row must never re-derive ranges and never read `dataset` ranges.
- The whole row is one native `button` (`type="button"`), whose accessible name comes from its content and therefore includes the building name. Its content: name (`font-medium`, one line, `title` with the full name), district and province
  joined with ` · ` (skip a `null` district without printing `null`), `countLabel(building.unitCount)`, `priceRange(building)`, and the maximum size via `sqm`.
- `selected` sets `aria-current="true"` and the accent border (`#0f766e` / the token from 05-01's theme); unselected has no `aria-current` and the default border (`#e5e7eb`).
- Clicking calls `onSelect(building.id)`. No expansion, no chevron, no units here (06-08); keep the button free of interactive descendants so that task can add them as siblings.
- One-sentence comment: "summary row for a recomputed building; selection only".

Run `pnpm -C web test --run BuildingRow` — seven green.

## Refactor / keep it DRY
`countLabel`, `priceRange`, `sqm` are imported, never re-implemented inline (a second range formatter would drift from `format.ts`). The district/province join is one expression, not a helper used once. Keep the class strings literal and
consistent with the tokens; 08-08 checks contrast.

## Verification (how to test it)
```text
pnpm -C web test --run BuildingRow
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: 7 passed; whole web suite green; typecheck clean; `make check` exits 0.

## Out of scope (do not do)
- Units inside the row, chevron, expansion state (06-08).
- The list container, loading/empty states, counts header (06-09, 06-10).
- Selection state ownership (`App`/`useSelection`), scroll-into-view, hover tooltips, map sync.
- Any sorting: input order is the display order.

## Definition of done
- [ ] Seven cases exist, named as written, and pass.
- [ ] Red run observed and quoted in the commit body.
- [ ] Web suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the two files above.
- [ ] `status: done`; `## Result` filled; one commit on `main`.

## Commit
```text
06-06: add the building row with selection

- points: 1
- red: test_row_shows_name_district_count_and_ranges failed (Failed to resolve import "./BuildingRow")
- add BuildingRow as a single button with name, district, count, price range and max size
- selected rows set aria-current and the accent border; long names truncate with a title
- suite: 7 passed
```

Commit this task alone, with the listed files only, on the trunk (`main`) — one task = one
commit, no branch, no push.

## Traps and review notes
- Building the row as `div role="button"` loses Space/Enter and focus styling; case 7 guards it.
- Printing the dataset's `priceMin` in a filtered list is the W-12 failure; the props contract says the object is already recomputed, and the review note is that the component must not accept a dataset.
- Joining `district` blindly prints `null` for a district-less building; test with the sample's nullable case.
- `aria-current="false"` is not the same as absent; case 5 pins the absence.
- Wrapping long names to four lines breaks the 64 px row rhythm — class assertion plus `title` is the check, not computed CSS.
- Reviewer greps for `onClick` on a non-button, for local `toFixed`/range formatting, and for any `building.units.map` (units belong to 06-08).

## Plan samples are untrusted
The plan shows the row anatomy as a text diagram, not markup. Verify the formatter names and the theme token names against `web/src/lib/format.ts` and `tailwind.config.ts` (`CLAUDE.md` §6); record deviations under `## Decisions`.

## Decisions
## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
