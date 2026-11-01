---
id: 06-08
stage: 06
title: Expandable building row reveals its units
points: 1
owner: web
status: todo
depends_on: [06-06, 06-07]
files:
  - web/src/components/BuildingRow.tsx
  - web/src/components/BuildingRow.test.tsx
---

# 06-08 · Expandable building row reveals its units
**Points:** 1 · **Stage:** 06 — Filters and results list · **Owner lane:** web · **Depends on:** 06-06, 06-07 · **Unblocks:** nothing

## Goal
Make `BuildingRow` expandable: a chevron toggle (collapsed by default, Enter/Space operable) reveals the building's units through `UnitRow`, with expansion state local to the row and the unit links rendered outside the selectable button.

## Why this task exists
The product's point is grouping by building, but a visitor still needs the individual listings without leaving the list. The plan's answer is one extra affordance: expand a row in place. Two contracts are easy to get wrong and both are
tested here — the row must remain one selection control with a separate toggle (a nested button is invalid HTML and breaks keyboard navigation), and the unit links must live outside the selection button, so clicking "view ↗" opens the
listing instead of selecting the building. Expansion is deliberately local: two rows expand independently and `App` never learns about it.

## Current state (preconditions)
- 06-06 done: `BuildingRow` renders one native button with `aria-current` when selected and calls `onSelect(id)`; its seven cases pass and none asserts that the row has exactly one control.
- 06-07 done: `UnitRow` renders one unit line plus the safe external link and is not a button.
- `Building` objects arriving here are already recomputed for the current filters (contract from 06-06).

## Read before you start
- `plan/stage-06-filters.md` Task 6.3 items 3, 4 and the Traps entry about the link click being swallowed.
- `plan/02-architecture.md` §5.1 ("Expanding a row shows its units inline — same component as the left panel list; one extra affordance, no extra screen").
- `plan/90-test-matrix.md` rows W-09 (selection) and W-13 (link safety) — this task must not regress them.
- `web/src/components/BuildingRow.tsx` and `UnitRow.tsx` — the current markup you are extending.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/components/BuildingRow.tsx` | chevron toggle, local expansion state, unit list |
| modify | `web/src/components/BuildingRow.test.tsx` | the seven cases below; keep the 06-06 cases green |

Nothing else may change: no `ResultsList`, no `App.tsx`, no `UnitRow` edits.

## Test first (red)
1. `test_collapsed_by_default_and_renders_no_units` — a row for a building with two units shows a button named `/show units/i` with `aria-expanded="false"`; no unit id and no `view` link is in the document.
2. `test_clicking_the_chevron_expands_and_renders_unit_rows` — after the click the toggle reads `/hide units/i` with `aria-expanded="true"`, both unit ids are visible, and each has a `view` link (proving `UnitRow` is reused, not re-marked-up).
3. `test_enter_on_the_chevron_expands_the_row` — focus the toggle, press Enter (and, in a second interaction, Space): each expands; the toggle is reachable by Tab while collapsed.
4. `test_expanded_units_live_outside_the_selection_button` — for every rendered `view` link, `closest("button")` is `null`, and clicking a unit link does not call `onSelect`.
5. `test_toggling_does_not_select_and_selecting_does_not_toggle` — clicking the chevron never calls `onSelect`; clicking the row's info button calls `onSelect` once and leaves `aria-expanded` unchanged.
6. `test_expansion_state_is_local_to_each_row` — with two rows mounted, expanding the first leaves the second collapsed; re-rendering the first with a different `selected` value keeps it expanded.
7. `test_first_unit_link_is_focusable_only_when_expanded` — collapsed: the row contains no link at all; expanded: Tab from the toggle reaches the first unit's `view` link.

Expected red: `Unable to find role="button" and name /show units/i`. Run `pnpm -C web test --run BuildingRow`, quote one line.

## Implement (green)
Extend `BuildingRow`'s props with nothing new — `building`, `selected`, `onSelect` stay; expansion is a local `useState` initialised collapsed.

- Anatomy: one container (a list item when inside the list) holding two sibling controls: the existing info button (name, district, counts, ranges, `aria-current`, `onSelect`) and a chevron toggle button labelled `Show units` / `Hide units`
  (include the building name if you prefer, but the tests match the verb phrase) with `aria-expanded` and `aria-controls` pointing at the units container's id.
- The units container sits after both buttons, inside the same list item, and renders one `UnitRow` per unit in the order given (the pipeline sorts by price then id; do not re-sort). It is not rendered at all while collapsed.
- The chevron's click must not call `onSelect`; the info button's click must not toggle. Do not put one control inside the other.
- Accessible names: the toggle's name must change with state; the info button's name must keep containing the building name (06-06 case 2 still runs).
- One-sentence comment: "row selects; a separate chevron expands, state local to the row".

Run `pnpm -C web test --run BuildingRow` — fourteen green.

## Refactor / keep it DRY
`UnitRow` is imported, never copied. The units container id is generated per row (not global), so two expanded rows cannot point `aria-controls` at the same node. No expansion callback leaks to the parent, no context, no store (YAGNI).

## Verification (how to test it)
```text
pnpm -C web test --run BuildingRow
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: 14 passed in the target file; the 06-06 cases still pass unedited; suite and typecheck clean.

## Out of scope (do not do)
- The list container, loading/empty states (06-09); counts and partial warning (06-10); perf (06-11).
- Lifting expansion to `App`, "expand all", remembering expansion across filter changes, animation.
- The left panel and map selection sync (stage 07); deep links to an expanded row.

## Definition of done
- [ ] Seven cases exist, named as written, and pass; 06-06 and 06-07 cases untouched and green.
- [ ] Red run observed and quoted; case 7 asserts absence before expansion, not just presence after.
- [ ] Web suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the two files above.
- [ ] `status: done`; `## Result` filled; one commit on `main`.

## Commit
```text
06-08: expand building rows to reveal units

- points: 1
- red: test_clicking_the_chevron_expands_and_renders_unit_rows failed (Unable to find role="button" and name /show units/i)
- add a chevron toggle with aria-expanded and a local expansion state
- render units through UnitRow below the row button, outside any clickable ancestor
- suite: 14 passed
```

Commit this task alone, with the listed files only, on the trunk (`main`) — one task = one
commit, no branch, no push.

## Traps and review notes
- Nesting the chevron inside the row button (or units inside it) is invalid HTML and makes the "view" link select the building; case 4's `closest("button")` check is the guard.
- Expansion in `App` state makes filter changes collapse/expand rows at random; it is local by design — case 6 pins it.
- `aria-expanded` on the wrong element (the container instead of the toggle) reads as an unlabelled control; keep it on the button whose name changes.
- Rendering units while collapsed and hiding them with CSS leaves links tabbable and breaks case 7; do not render them until expanded.
- Reviewer greps for `useState` in `App` near expansion, for a copied unit line, and for a hardcoded `aria-controls` id.

## Plan samples are untrusted
The plan's anatomy block shows an untruncated row; it does not specify where the chevron lives. Verify the React version's `useId` availability and testing-library role queries against `web/package.json` (`CLAUDE.md` §6); record deviations
under `## Decisions`.

## Decisions
## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
