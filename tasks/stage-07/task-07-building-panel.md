---
id: 07-07
stage: 07
title: Left building panel with placeholder
points: 1
owner: web
status: todo
depends_on: [05-12]
files:
  - web/src/components/BuildingPanel.tsx
  - web/src/components/BuildingPanel.test.tsx
---

# 07-07 · Left building panel with placeholder

**Points:** 1 · **Stage:** 07 — Map and selection · **Owner lane:** web · **Depends on:** 05-12 · **Unblocks:** 07-08

## Goal

Create the left panel for a selected building — name, district · province, unit count, price range, size range and a close control — with an instructional placeholder instead of an empty box when nothing is selected.

## Why this task exists

The layout reserves 340 px on the left for "the units of the selected building" (`plan/02-architecture.md` §5.1), and the panel is one of the three places selection must appear (W-09). It is also the first screen a visitor sees before selecting anything: an empty bordered card reads as a broken feature, so the null state is an explicit instruction. All the numbers here are already computed by `groupByBuilding` for the current filters — the panel is a presenter and must not recompute ranges, or a filtered panel would advertise prices of units that are not shown. Units, the maps link and the no-location text follow in 07-08.

## Current state (preconditions)

- 05-10 done: `lib/group.ts` exports `countLabel(n)` (`3 units`) and `priceRange(building)` (`45–50k`); 05-12 done: `lib/format.ts` exports `sqm(n)` (`65 m²`).
- 05-03 done: the sample fixture's `p00348` is `The Bangkok Sathorn`, district `Sa Thon`, province `Bangkok`, `unitCount` 3, prices 45000/50000, sizes 58/65.
- 05-18 done by lane order: the layout has the 340 px left column; 05-17 done: `App` renders a left placeholder panel. `App.tsx` is **not** touched by this task.
- `web/src/components/BuildingPanel.tsx` and its test do not exist.

## Read before you start

- `CLAUDE.md` §3, §5 (DRY: formatters are imported; no new helper for one caller), §6.
- `plan/02-architecture.md` §5.1 — the left panel block, its 340 px column and the row anatomy; §5.3 — tokens (14 px base, `font-medium` names, `tabular-nums` prices, `shadow-sm` only here and on the header).
- `plan/stage-07-map.md` Task 7.3 items 1–2 and item 4 — placeholder copy, panel content, close button.
- `plan/90-test-matrix.md` row W-09 — the panel half of selection sync.
- `web/src/lib/group.ts` and `web/src/lib/format.ts` — the exact formatter signatures this panel composes.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `web/src/components/BuildingPanel.tsx` | the panel, its props and the placeholder |
| create | `web/src/components/BuildingPanel.test.tsx` | the seven cases below |

Nothing else may change: no `App.tsx`, no units list (07-08), no `UnitRow`, no `lib/` edit.

## Test first (red)

Build the building from the fixture (`dataset.sample.json`) and override only the fields a case exercises.

1. `test_null_building_shows_the_placeholder_instruction` — `building: null` renders exactly `Select a building on the map or in the list`, has no `h2`, and renders no card container or close button (an empty box is the bug this case prevents).
2. `test_selected_building_name_is_a_second_level_heading` — `getByRole("heading", { level: 2, name: "The Bangkok Sathorn" })` resolves; the name is not a button or a link.
3. `test_location_line_joins_district_and_province` — the line reads `Sa Thon · Bangkok` (middle dot with a space on each side); a `district: null` building renders only `Bangkok` and never the string `null`.
4. `test_summary_shows_unit_count_and_price_range` — the panel contains `3 units` and `45–50k` with the en dash U+2013, taken from `countLabel`/`priceRange`, not re-typed.
5. `test_size_range_uses_the_shared_area_formatter` — `sizeMin` 58 / `sizeMax` 65 renders `58–65 m²`; a building with equal bounds renders `65 m²` once, not `65–65 m²`.
6. `test_close_button_is_named_clear_selection_and_calls_on_close` — `getByRole("button", { name: "Clear selection" })` resolves (a visible × may carry that `aria-label`), is a native button, and one click calls `onClose` exactly once.
7. `test_selected_building_never_renders_the_placeholder` — with a building, the instruction text is absent and the panel has exactly one close control.

Expected red: `Failed to resolve import "./BuildingPanel"`. Run `pnpm -C web test --run BuildingPanel`, quote one line.

## Implement (green)

- Props exactly two: `building` (a `Building`, already recomputed for the current filters, or `null`) and `onClose` (a no-argument callback). No `units` prop, no other callbacks, no selection state.
- `building === null` → one paragraph with the exact instruction text, styled as muted secondary text. No wrapper card, no heading, no close button, no skeleton.
- With a building (a rebuilt `Building` from `groupByBuilding`, so its ranges already reflect the filters): `h2` with the name; a location line joining district and province with ` · `; `countLabel(building.unitCount)`; `priceRange(building)`; the size range composed from `sqm` with the same en-dash rule as `priceRange` (one expression, not a helper used once).
- Header block only: the units list is 07-08. Leave a clearly marked place for it rather than inventing markup.
- Close: a native `button` whose accessible name is exactly `Clear selection`, calling `onClose` once; no navigation, no other side effect.
- Layout: the panel fills the 340 px column the layout gives it; `shadow-sm` is allowed here (the design permits it on the selected panel). No pixel widths inside the component.
- One-sentence comment: "left panel for the selected building; a null building is an instruction, not an empty card".

Run: `pnpm -C web test --run BuildingPanel` — seven green.

## Refactor / keep it DRY

`countLabel`, `priceRange` and `sqm` are imported; no local pluraliser, no `toLocaleString`, no second en-dash table. The district/province join is one expression (06-06 does the same in a row — if that expression already exists as a helper, import it; otherwise do not create a helper for two call sites that differ in layout). The panel is a presenter: it must not import `lib/filter.ts`, `lib/group.ts`'s grouping function, or the dataset.

## Verification (how to test it)

```text
pnpm -C web test --run BuildingPanel
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Manual check: after 07-09, `pnpm -C web dev` → the left column shows the instruction before any selection, then the building's name, location and ranges; the × clears it; no layout shift when the panel fills.

## Out of scope (do not do)

- The units list, `UnitRow`, the maps link and the no-location text — 07-08.
- Wiring into `App`, selection state, scroll-into-view, keyboard shortcuts (Escape to close is 07-12/08-07).
- Editing, comparing or filtering units; a "similar buildings" section; images.
- A generic `Panel`/`Card` component with one caller.

## Definition of done

- [ ] All seven cases exist, named as written, and pass.
- [ ] Red run observed and quoted; the placeholder case asserts the absence of a card and a heading, not only the presence of text.
- [ ] The exact strings (`Sa Thon · Bangkok`, `3 units`, `45–50k`, `58–65 m²`, `Clear selection`) are asserted, en dashes included.
- [ ] Lane suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the two files above plus this task file.
- [ ] `status: done`; `## Result` filled; one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
07-07: add the left building panel with a placeholder

- points: 1
- red: test_null_building_shows_the_placeholder_instruction failed (Failed to resolve import "./BuildingPanel")
- add BuildingPanel with name, district and province, count, price and size ranges
- placeholder instruction instead of an empty card, close named Clear selection
- suite: N passed
```

Files in the commit: exactly the two listed files plus this task file.

## Traps and review notes

- A card with nothing in it is the most common version of this bug; case 1 checks the absence of the card and the heading.
- Re-typing `units`/`k` formatting in the panel drifts from the row strings on the first change; the exact assertions catch it.
- The close control with a bare × has no accessible name; the `aria-label` is the contract and case 6 asserts the name, not the glyph.
- Reading `dataset` inside the panel (instead of the passed building) makes the panel disagree with the list after filtering.
- Rendering the panel's units block "while you are here" would collide with 07-08's tests; leave the place empty.
- Reviewers: grep `BuildingPanel.tsx` for `toLocaleString`, `toFixed`, `Math.` and a second `unitCount` computation (none).

## Plan samples are untrusted

The plan's panel sketch is a text diagram. Verify `Building`'s field names (`district`, `province`, `sizeMin`, `sizeMax`, `unitCount`), the formatter signatures and the Tailwind tokens in the repo before using them (`CLAUDE.md` §6). Record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
