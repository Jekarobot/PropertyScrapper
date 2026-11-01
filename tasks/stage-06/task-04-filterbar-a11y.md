---
id: 06-04
stage: 06
title: Reset, live result count, labels and tab order
points: 1
owner: web
status: todo
depends_on: [06-03]
files:
  - web/src/components/FilterBar.tsx
  - web/src/components/FilterBar.test.tsx
---

# 06-04 · Reset, live result count, labels and tab order
**Points:** 1 · **Stage:** 06 — Filters and results list · **Owner lane:** web · **Depends on:** 06-03 · **Unblocks:** 06-05

## Goal
Finish the filter bar: a Reset button disabled while the filters equal the defaults, a polite live region announcing the result count, a named control group, an accessible name on every control, and the documented Tab order.

## Why this task exists
The bar is now functionally complete but not usable: a keyboard user cannot tell where the controls begin, a screen-reader user does not hear that the list shrank when a filter changed, and there is no way back to the full list. Reset also
has a subtle contract — it must be inert when nothing is filtered, otherwise it invites a click that does nothing. This task closes W-11 and produces the exact interface 06-05 wires: `onReset` and `resultCount` finally have a consumer.

## Current state (preconditions)
- 06-01 to 06-03 done: size pair, bedrooms, district and building controls render; a render helper and a `FilterBarProps` fixture helper exist in `FilterBar.test.tsx`.
- `FilterBarProps` already declares `resultCount: number` and `onReset: () => void`; nothing consumes them yet.
- `useFilters` defaults are `null`, `null`, `"any"`, `"all"`, `""`.

## Read before you start
- `plan/stage-06-filters.md` Task 6.1 items 8–10 and "Accessibility specifics" (fieldset/legend acceptable, no visually hidden label without an `aria-label`).
- `plan/02-architecture.md` §5.1 (the count sits at the right end of the bar, `12 found`), §5.3 tokens.
- `plan/90-test-matrix.md` rows W-11 and W-17 (keyboard reachability, live regions).
- `plan/stage-07-map.md` (skim only) — 07-12 reuses this bar in the mobile sheet, so the group name and Tab order must not be viewport-specific.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/components/FilterBar.tsx` | reset button, live count region, group name, ordering |
| modify | `web/src/components/FilterBar.test.tsx` | the seven cases below |

Nothing else may change: no `App.tsx`, no `EmptyState`, no new file.

## Test first (red)
1. `test_reset_is_disabled_when_filters_equal_the_defaults` — the button named `Reset` is disabled for the default filters; it is a `button`, not a link.
2. `test_reset_is_enabled_when_any_single_filter_differs` — table over five variants (`sizeMin` 60, `sizeMax` 120, `beds` `"studio"`, `district` `"Watthana"`, `building` `"x"`): each alone enables it, proving the comparison covers all five
   keys.
3. `test_clicking_reset_calls_on_reset_once` — with a non-default filter, click reports `onReset` exactly once and does not emit `onChange`.
4. `test_result_count_is_a_polite_live_region` — `resultCount` 12 renders `12 found` inside an element with `aria-live="polite"`; 1 renders `1 found`; 0 renders `0 found`; the region exists in all three cases (it is not conditionally mounted).
5. `test_controls_are_grouped_under_an_accessible_group_name` — `getByRole("group", { name: /search filters/i })` contains all five controls and the Reset button: `fieldset` + visually hidden legend, or an explicit `role="group"` with
   `aria-label`.
6. `test_every_control_has_an_accessible_name` — `getByLabelText` resolves `Size minimum`, `Size maximum`, `Bedrooms`, `District`, `Building`, and `getByRole("button", { name: "Reset" })` resolves; no control relies on a placeholder for its
   name.
7. `test_tab_order_is_min_max_beds_district_building_reset` — starting from the body with non-default filters, `userEvent.tab()` visits exactly that sequence; Reset is reachable last because it is enabled.

Expected red: `Unable to find role="button" and name "Reset"`. Run `pnpm -C web test --run FilterBar`, quote one line.

## Implement (green)
- Reset: a `button` labelled `Reset` (an icon may accompany it; the accessible name stays `Reset`), `disabled` exactly when the five filter values equal the defaults. Prefer importing the defaults from `useFilters` if that module exports
  them; otherwise one local predicate in this file with a comment saying it mirrors `useFilters` — never a second copy of the object in `App`.
- Count: `resultCount` is the number of **units** passing the filters (not buildings) — same number the 06-10 header later splits into `N buildings · M units`. Render `N found`, singular `1 found`, inside a persistent `aria-live="polite"`
  element (`aria-atomic` true so the whole phrase is read). Do not announce the count elsewhere.
- Grouping: one named group around all controls and Reset, with the accessible name `Search filters` (exact words, any case). Do not hide labels visually without an accessible name.
- Order in the DOM is the Tab order: minimum, maximum, bedrooms, district, building, reset. No positive `tabIndex` values anywhere.
- Reuse the 06-01/06-03 helpers; no new state beyond what the count/reset need (both come from props).

Run `pnpm -C web test --run FilterBar` — twenty-six green.

## Refactor / keep it DRY
One default-comparison predicate, one live region. If `useFilters` exports its defaults, delete the local copy rather than keeping both. Keep the count as a pure prop render: no counting logic in the component.

## Verification (how to test it)
```text
pnpm -C web test --run FilterBar
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: 26 passed in the target file; suite and typecheck clean; `make check` exits 0.

## Out of scope (do not do)
- Wiring into `App.tsx`, computing `resultCount`, or calling `reset()` (06-05).
- The results-list header (`N buildings · M units`, 06-10), the empty state (06-09/06-12).
- Focus trapping, roving tabindex, skip links, mobile sheet layout (07-12), contrast work (08-08).

## Definition of done
- [ ] Seven cases exist, named as written, and pass; all earlier `FilterBar` cases still pass.
- [ ] Red run observed and quoted; the Tab-order assertion compares the real active element sequence.
- [ ] Web suite and typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the two files above.
- [ ] `status: done`; `## Result` filled; one commit on `main`.

## Commit
```text
06-04: finish filter bar reset, count and accessibility

- points: 1
- red: test_clicking_reset_calls_on_reset_once failed (Unable to find role="button" and name "Reset")
- add a Reset button disabled on default filters, a polite live result count
- add a named group and pin the Tab order min, max, beds, district, building, reset
- suite: 26 passed
```

Commit this task alone, with the listed files only, on the trunk (`main`) — one task = one
commit, no branch, no push.

## Traps and review notes
- A live region that is mounted only when the count changes announces nothing; keep it always present.
- `12 found` versus `12 results` versus `12 units found` — the exact string is asserted, so pick the plan's wording and do not embellish.
- Enabling Reset by comparing only one key is the classic bug; the five-variant table exists for it.
- Disabled buttons are skipped by Tab, so the Tab-order test must run with an enabled Reset; a green run with defaults would prove nothing.
- Do not put `aria-live` on the whole bar: every keystroke in the building field would re-announce the layout.
- Reviewer greps `tabIndex=`, `aria-live` occurrences (exactly one), and any `reset(` call inside the component.

## Plan samples are untrusted
The plan gives the count format and the fieldset hint, no markup. Verify `getByRole("group")` semantics for `fieldset` in the installed testing-library version (`CLAUDE.md` §6) before assuming it; record deviations under `## Decisions`.

## Decisions
## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
