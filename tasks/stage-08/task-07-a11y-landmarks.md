---
id: 08-07
stage: 08
title: Landmarks, focus and live announcements
points: 1
owner: web
status: todo
depends_on: [07-09]
files:
  - web/src/App.tsx
  - web/src/App.test.tsx
  - web/src/a11y.test.tsx
---

# 08-07 · Landmarks, focus and live announcements

**Points:** 1 · **Stage:** 08 — Hardening and release · **Owner lane:** web · **Depends on:** 07-09 · **Unblocks:** 08-09

## Goal

Make the composed app navigable and legible to a keyboard and a screen reader: one `h1`, five named landmarks, a focus ring on every control, no positive tab index, and one live region that announces the selection.

## Why this task exists

Stages 05–07 built five regions and an interactive map without auditing the finished tree: each component was tested alone, so nobody has proved that the composed page has exactly one `h1`, that the map and results sections are distinguishable
when a screen reader lists regions, or that the filters can be operated without a mouse. The count is already announced by `FilterBar`'s live region (06-04), but a selection is not: a keyboard user expands a row, the left panel fills, and
nothing is spoken. This is the W-17 row of the matrix and the accessibility half of the mobile pass (08-09), where the same controls must stay reachable behind a button. No behaviour changes, no redesign, no new component.

## Current state (preconditions)

- 05-18 is done: `App` renders `header`, `main`, `aside` and `section[aria-label="Results"]`; `StatusBar` renders the freshness line and the partial banner.
- 06-04 is done: `FilterBar` groups its controls under an accessible name, labels each control, keeps Reset last in the tab order, and renders `N found` inside a persistent `aria-live="polite"` element.
- 06-06/06-09 are done: rows are focusable selection targets carrying `aria-current` when selected.
- 07-09 is done: one selection is shared by the map, the list and the panel; empty map click or the panel's close clears it.
- `web/src/App.test.tsx` exists with a render helper that stubs `fetch` with `dataset.sample.json` and mocks Leaflet. `pnpm -C web test --run` and `pnpm -C web typecheck` are green.

## Read before you start

- `CLAUDE.md` §3 (component tests in jsdom, no network), §5 (one live region, no duplicate logic), §6, §7.
- `plan/02-architecture.md` §5.1 (the five regions and what each holds) and §5.2 F9/F11.
- `plan/stage-08-hardening-ops.md` Task 8.4 items 1–3 — the landmark list and the "focus-visible class assertion is acceptable" allowance.
- `plan/stage-05-web-shell.md` Task 5.5 (the skeleton that created the landmarks); `plan/90-test-matrix.md` row W-17; `web/src/components/FilterBar.tsx` (the existing live region).

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/App.tsx` | the single `h1`, the Map section, the selection status region |
| modify | `web/src/App.test.tsx` | the two composed-app cases below |
| create | `web/src/a11y.test.tsx` | the five structural and keyboard cases below |

Nothing else: no `FilterBar.tsx`, no `ResultsList.tsx`, no `MapView.tsx`, no CSS, no dependency (no axe-core, no jest-axe).

## Test first (red)

Use the existing render helper; every case renders the whole `App` with the sample dataset, because these are properties of the composition, not of a component.

1. `a11y.test.tsx::test_exactly_one_h1_and_it_lives_in_the_banner` — exactly one `h1`, inside the `banner` landmark; assert both the length and the containment.
2. `::test_five_named_landmarks_are_present` — `banner`, `main`, `complementary`, `region` named `Results`, `region` named `Map`; exactly one `main`; the Results region contains the list and the Map region contains the (mocked) map container.
   Unnamed sections are invisible to a screen reader's region list.
3. `::test_every_control_is_reachable_with_the_keyboard_only` — tab from the document body until focus wraps (hard cap), collecting every focused element: the five filter controls, `Reset`, the three building rows and the map/list affordance
   must all appear. No mouse events in this case.
4. `::test_no_element_has_a_positive_tabindex` — every element with a `tabindex` parses to 0 or less; Leaflet's `-1` and `0` are allowed, a positive value is not.
5. `::test_every_app_control_carries_a_visible_focus_ring` — each control from case 3 has the `focus-visible:ring-2` utility in its class list; this is the assertion the plan sanctions instead of a computed outline jsdom cannot produce.
6. `App.test.tsx::test_selection_is_announced_in_one_status_region` — after selecting a building, one `role="status"` region contains its name and unit count; after deselecting it is empty (or says the panel is closed) and there is still exactly
   one selection region. `N found` stays owned by `FilterBar` — assert it is not duplicated here.
7. `App.test.tsx::test_keyboard_only_filter_flow_updates_the_results` — focus the district control, change it with the keyboard, assert the results header count changes and focus is still inside the filter group; no `click` on any control.

Expected red: `Unable to find role="status"` on case 6 and a missing `Map` region or `expected 2 to be 1` on cases 1–2. Run `pnpm -C web test --run a11y` and quote one line.

## Implement (green)

- Shell: `header` with the single `h1` (product name; every other heading is `h2`), `main` containing `aside` and `section[aria-label="Results"]`, and a `section[aria-label="Map"]` around the map component. The map section is a sibling of
  `main`, not a child of the results section: it spans the full width and nesting would announce the map as part of the list.
- Selection announcement: one `role="status"` element with `aria-live="polite"`, rendered unconditionally so it exists before the first selection, holding one phrase such as `<name> selected, <n> units` and an empty string when nothing is
  selected. Place it next to the panel, not in the filter bar.
- Do not add a second count region: `FilterBar` already announces `N found`; App must not repeat it. A viewport count, if needed, is visual text outside any live region.
- Focus: no positive `tabindex`; rows stay native buttons or links; the map's keyboard affordances come from Leaflet and are not re-implemented. Every control App renders keeps `focus-visible:ring-2`; no restyling.
- Keep the existing props and state: this adds markup and one derived string, not state, an effect or a context. One-sentence comment above the status region stating what it announces and when.

Run: `pnpm -C web test --run a11y` and `pnpm -C web test --run App` — all new cases green.

## Refactor / keep it DRY

The selection phrase is derived once in `App` from the building it already computes — do not recount units in the region, and do not add a second formatter: `countLabel` from `lib/group.ts` (05-10) is the label. The region names `Results` and
`Map` are literals in one file each; if `ResultsList` already owns the Results wrapper, keep it there and assert it instead of nesting a new section.

## Verification (how to test it)

```text
pnpm -C web test --run a11y
pnpm -C web test --run App
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: the new cases pass; the web suite and typecheck are green; `make check` exits 0. Manual check: Tab from the address bar and see the ring on every stop; select a row by keyboard and confirm (with a screen reader or the accessibility
pane) one announcement and five named landmarks.

## Out of scope (do not do)

- An accessibility linter dependency, a skip link, a focus trap, a custom keyboard map handler, RTL, high-contrast theming, or aria attributes beyond what Leaflet ships.
- Renaming or restructuring components, moving selection state, or adding a second live region.
- Visual changes beyond the existing focus ring; contrast is 08-08, mobile layout 08-09, motion 08-10.
- Editing `FilterBar`, `ResultsList`, `BuildingRow`, `BuildingPanel` or their tests.

## Definition of done

- [ ] All seven cases exist, named as written, and pass.
- [ ] Red run observed and quoted in the commit body.
- [ ] One `h1`, five named landmarks, one selection status region, no positive `tabindex`.
- [ ] `pnpm -C web test --run` and `pnpm -C web typecheck` green; `make check` exits 0.
- [ ] Only the three listed files and this task file changed (`git status --short`).
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
08-07: add accessible landmarks and selection announcement

- points: 1
- red: test_selection_is_announced_in_one_status_region failed (Unable to find role="status")
- add the single h1, the named Map region and one polite selection status region
- assert landmark structure, keyboard reachability, focus ring and no positive tabindex
- suite: 41 web tests
```

Files in the commit: `web/src/App.tsx`, `web/src/App.test.tsx`, `web/src/a11y.test.tsx`, this task file. One commit, on trunk `main`.

## Traps and review notes

- `getByRole("region")` only finds sections with an accessible name; a `section` without `aria-label` silently disappears from the query and the test passes for the wrong reason. Assert the named-region count too.
- A live region mounted only when something is selected is never announced reliably: the element must exist before the first change.
- Tab-order tests that stop after ten `tab()` calls pass on a page with fewer controls; loop until focus wraps, with a hard cap, and assert the set.
- Adding `aria-live` to the existing count element duplicates the announcement with `FilterBar`; check the composed tree for two live regions before committing.
- Reviewers: `grep -n 'aria-live' web/src` finds one in `FilterBar.tsx` and one in `App.tsx`; `grep -rn 'tabIndex' web/src` has no positive literal; `grep -c '<h1' web/src/App.tsx` is 1.

## Plan samples are untrusted

Task 8.4 mixes Tailwind-class assertions with behaviour and does not say where the Map region lives; it also predates the `FilterBar` live region from 06-04. Verify each assertion against the installed Testing Library version and the composed
tree, and record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |

