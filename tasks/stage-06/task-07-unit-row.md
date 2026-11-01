---
id: 06-07
stage: 06
title: Shared unit row with safe external links
points: 1
owner: web
status: todo
depends_on: [05-12]
files:
  - web/src/components/UnitRow.tsx
  - web/src/components/UnitRow.test.tsx
---

# 06-07 · Shared unit row with safe external links
**Points:** 1 · **Stage:** 06 — Filters and results list · **Owner lane:** web · **Depends on:** 05-12 · **Unblocks:** 06-08

## Goal
Create the one `UnitRow` markup used by both the expanded building row and the stage-07 building panel: unit id, bedrooms, size, price, and a `view ↗` link that opens the listing in a new tab with `rel="noopener noreferrer"`.

## Why this task exists
A unit line appears in two places (the inline expansion of a list row and the left panel), and `CLAUDE.md` §5 names `components/UnitRow.tsx` as the single implementation of that markup — two copies would drift in price formatting and,
worse, in link safety. The link is the only way out of the app to the real listing, it always opens a new tab, and the missing `noopener` would give the third-party page a handle on our window. The link also must not sit inside the
selectable row button, or clicking "view" would select the building and (on some browsers) swallow the navigation.

## Current state (preconditions)
- 05-12 done: `lib/format.ts` exports `thb(n)` → `฿45,000`, `sqm(n)` → `65 m²`, `beds(n)` → `Studio` / `1 bed` / `2 beds`.
- 05-03 done: the `Unit` type has `id`, `url`, `priceThb`, `sizeSqm`, `bedrooms`, `bathrooms`, `floor` (`number | null`), `updatedOn`; the fixture's units carry real values to assert with.
- `BuildingRow` (06-06) exists and does **not** render units yet.
- `web/src/components/UnitRow.tsx` and its test do **not** exist.

## Read before you start
- `plan/stage-06-filters.md` Task 6.3 item 4 and the Traps entry about the link's click being swallowed.
- `plan/02-architecture.md` §5.1 ("The unit link always opens the site in a new tab: `target="_blank" rel="noopener noreferrer"`") and the row anatomy block (`CX-84121  1 bed · 65 m² · fl 27   ฿45,000   [view ↗]`).
- `plan/90-test-matrix.md` row W-13 — all three attributes must be asserted.
- `web/src/lib/format.ts` — the three formatters; sizes are decimals, so do not round locally.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `web/src/components/UnitRow.tsx` | the unit line and the safe link |
| create | `web/src/components/UnitRow.test.tsx` | the six cases below |

Nothing else may change: no `BuildingRow` wiring yet (06-08), no `App.tsx`, no `lib/` edits.

## Test first (red)
Build the unit props from the fixture values; write the expected strings as literals.

1. `test_renders_id_size_price_and_bedrooms` — a unit with id `CX-84121`, 65 sqm, ฿45 000, 1 bedroom renders all four (`65 m²`, `฿45,000`, `1 bed`, the id); the string `undefined`/`NaN` appears nowhere.
2. `test_studio_unit_shows_studio_instead_of_zero_beds` — `bedrooms: 0` renders `Studio` and not `0 bed`/`0 beds`.
3. `test_null_floor_is_omitted` — `floor: null` renders no floor fragment and no literal `null`; a real floor (`27`) renders `fl 27`.
4. `test_view_link_points_at_the_unit_url` — the link's `href` equals `unit.url` exactly (absolute `https://connex.in.th/...`, not a relative path).
5. `test_view_link_opens_in_a_new_tab_safely` — the link has `target="_blank"` **and** `rel="noopener noreferrer"` (assert the attribute string; both tokens required), and its accessible name contains `view`.
6. `test_link_click_does_not_bubble_and_is_not_inside_a_button` — rendered inside a parent whose click handler is a spy, clicking the link does not call the spy (propagation stopped), and `link.closest("button")` is `null`.

Expected red: `Failed to resolve import "./UnitRow"`. Run `pnpm -C web test --run UnitRow`, quote a line.

## Implement (green)
Export `UnitRow` with props `{ unit }` — no callbacks, no selection, no expansion:

- Line content in the plan's order: id, `beds(unit.bedrooms)`, `sqm(unit.sizeSqm)`, floor when present, `thb(unit.priceThb)` (with `tabular-nums`), then the link.
- Link: text `view ↗`, `href` the absolute `unit.url`, `target="_blank"`, `rel="noopener noreferrer"`, and a click handler that stops propagation so a surrounding clickable ancestor never treats a navigation as a selection.
- Markup: the row itself is not a button and contains no button; it is a plain container so it can be placed either outside the `BuildingRow` button or inside a non-interactive wrapper in the panel.
- Missing data: omit the floor fragment for `null`; do not render placeholders like `—` (YAGNI) and never print `null`/`NaN`.
- One-sentence comment: "one unit line, reused by the list and the panel; the link always opens a new tab".

Run `pnpm -C web test --run UnitRow` — six green.

## Refactor / keep it DRY
This file is the single implementation: `BuildingRow` (06-08) and `BuildingPanel` (07-08) import it and must not re-create the line. All number formatting goes through `lib/format.ts`; if a formatter is missing, extend `format.ts` in the
task that needs it with its own test — do not inline `toLocaleString`.

## Verification (how to test it)
```text
pnpm -C web test --run UnitRow
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: 6 passed; suite and typecheck clean; `make check` exits 0.

## Out of scope (do not do)
- Expansion, chevrons, the list container (06-08, 06-09).
- The left panel, map popups, "open on maps" (07-07/07-08).
- Hover previews, image thumbnails, relative dates, sorting units (the pipeline sorts by price then id).
- Any tracking/analytics on the outbound click.

## Definition of done
- [ ] Six cases exist, named as written, and pass.
- [ ] Red run observed and quoted; the `rel` assertion was not relaxed to "contains noopener".
- [ ] Web suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the two files above.
- [ ] `status: done`; `## Result` filled; one commit on `main`.

## Commit
```text
06-07: add the shared unit row with safe links

- points: 1
- red: test_view_link_opens_in_a_new_tab_safely failed (Failed to resolve import "./UnitRow")
- add UnitRow with id, beds, size, price, floor and the view link
- link opens a new tab with rel="noopener noreferrer" and stops propagation
- suite: 6 passed
```

Commit this task alone, with the listed files only, on the trunk (`main`) — one task = one
commit, no branch, no push.

## Traps and review notes
- `target="_blank"` without `noopener noreferrer` is a real vulnerability, not a style nit; the exact attribute is asserted.
- A relative `href` (from a stripped `url`) silently 404s on the app origin; case 4 compares the full string.
- React's `beds(0)` returning `Studio` is why the component must not print `bedrooms` directly.
- Putting the anchor inside the row button produces invalid nesting and a double action; the `closest` assertion catches it, and 06-08 must place this component as a sibling.
- Reviewer greps for `toLocaleString`, `toFixed`, `new Date`, and a second copy of the `view ↗` text.

## Plan samples are untrusted
The plan's anatomy line is a text diagram. Verify `beds`/`sqm`/`thb` signatures and the `Unit` type against `web/src/lib/format.ts` and `web/src/types.ts` (`CLAUDE.md` §6); record deviations under `## Decisions`.

## Decisions
## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
