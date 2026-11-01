# Stage 06 — Filters and the results list

**Goal:** the top filter bar and the right-hand list, wired to the Stage 05 state contract. No map
work here; the map placeholder stays.

**Depends on:** Stage 05 (merged contract). **Est:** 60 min. **Agent:** B. **Parallel with:** Stage 07
(different files, contract fixed by Stage 05).

**Files:**

```text
web/src/components/FilterBar.tsx        (+ FilterBar.test.tsx)
web/src/components/ResultsList.tsx      (+ ResultsList.test.tsx)
web/src/components/BuildingRow.tsx      (+ BuildingRow.test.tsx)
web/src/App.tsx                         (wire, no new logic)
web/src/components/EmptyState.tsx       (reuse; add "reset" variant)
```

---

## Task 6.1 — `FilterBar`

Controlled component, props:

```ts
interface FilterBarProps {
  filters: Filters;
  districts: string[];                       // sorted, from dataset
  buildingOptions: string[];                 // for the datalist, capped at 50 by relevance
  resultCount: number;
  onChange: <K extends keyof Filters>(key: K, value: Filters[K]) => void;
  onReset: () => void;
}
```

Layout (single row, wraps on small screens):

```
Size  [min] – [max] m²   |   Beds [Any ▾]   |   District [All ▾]   |   Building [____________]   (⟲ Reset)   12 found
```

**TDD order** (`FilterBar.test.tsx`, RTL + user-event):

1. Renders 4 labelled controls; every input has an accessible name (`Size minimum`, `Size maximum`,
   `Bedrooms`, `District`, `Building`). Query by `getByLabelText`, never by placeholder.
2. Typing `60` in the min field calls `onChange("sizeMin", 60)` **once** after the debounce
   (use `vi.useFakeTimers`, advance 200 ms). Clearing it calls `onChange("sizeMin", null)`.
3. Non-numeric input (`abc`) is rejected: the input shows the last valid value, no `onChange`.
4. `sizeMax < sizeMin` → the max field gets `aria-invalid="true"` and a hint appears
   (`role="alert"`: "Max size is below min size"); `onChange` still fires (the filter layer decides).
5. Beds options: `Any`, `Studio`, `1`, `2`, `3`, `4+` — selecting `4+` calls `onChange("beds", "4+")`.
6. District options come from props plus an `All` option; selecting one calls `onChange("district", …)`.
7. Building input has a `<datalist>` with at most 50 options and is debounced 200 ms; Enter does not
   submit anything (no form wrapper).
8. Reset button is disabled when `filters` equal the defaults; clicking it calls `onReset`.
9. `resultCount` is rendered in `aria-live="polite"` (`"12 found"` / `"1 found"`).
10. Keyboard: Tab order is min → max → beds → district → building → reset.

**Accessibility specifics:** `<fieldset>`/`<legend class="sr-only">Search filters</legend>` is
acceptable, but do not hide labels visually without an `aria-label`. Numbers use `inputMode="numeric"`
and `type="text"` (avoid `type=number` spinners).

Commit: `stage-06: filter bar component`.

---

## Task 6.2 — Filter wiring in `App`

**TDD order** (`App.test.tsx` additions):

1. With the sample dataset, `12 found`-style count reflects `filterUnits` (assert against
   `dataset.sample.json` numbers: unfiltered 6 units / 3 buildings).
2. Selecting `beds=2` leaves only the units with 2 bedrooms; buildings whose units all drop out
   disappear from the list, and their bubbles disappear in Stage 07 (assert the list here).
3. Setting `sizeMin=60` removes the 45 m² unit; the building row's price/size range is recomputed
   (F5).
4. `district=Watthana` keeps only `Premier Thonglor`.
5. Building text `bangkok` (lowercase) matches `The Bangkok Sathorn` (F4).
6. `reset()` restores the full list and clears `selectedId` (no ghost panel).

Commit: `stage-06: wire filters into app`.

---

## Task 6.3 — `BuildingRow` and `ResultsList`

Row anatomy (one per building, `role="row"` inside a `role="list"`):

```
▸ The Bangkok Sathorn              Sa Thon · Bangkok      3 units   45–50k   65 m²
   (expanded)
   CX-84121  1 bed · 65 m² · fl 27   ฿45,000   [view ↗]
```

**TDD order** (`BuildingRow.test.tsx`, `ResultsList.test.tsx`):

1. Row shows name, district, `countLabel(unitCount)`, `priceRange`, max size; the whole row is a
   `<button>` (keyboard reachable) whose accessible name includes the building name.
2. Clicking the row calls `onSelect(id)`; when `selected` is true it renders `aria-current="true"`
   and a visible accent border.
3. Expanding (Enter/click on the chevron) reveals its units; collapsed by default; expansion state is
   local to the row (`useState`), and the first unit link is focusable only when expanded.
4. Unit row: `CX-…` id, `beds()`, `sqm`, `thb`, and a link `view ↗` with
   `href=unit.url`, `target="_blank"`, `rel="noopener noreferrer"` (assert all three attributes — this
   is a hard requirement of the product).
5. `ResultsList` renders rows in dataset order; empty list → `EmptyState` with
   "No buildings in this area" + a "Clear filters" action; a `loading` skeleton when `status` is
   loading (3 grey rows, `aria-busy="true"`).
6. Long names truncate with `title` attr, never wrap to 4 lines (class assertion is acceptable).
7. With 200 buildings, the list renders < 100 ms (performance smoke: `performance.now()` around
   `render` with a generated dataset, budget 300 ms in CI).

Commit: `stage-06: results list and building rows`.

---

## Task 6.4 — Counts, empty and error states

1. `ResultsList` header shows `N buildings · M units` for the current filter+viewport set.
2. When filters produce 0 units but the dataset is non-empty → empty state with reset action, and the
   map (Stage 07) shows no bubbles.
3. When `dataset.stats.partial` → an inline warning above the list: "Data may be incomplete
   (partial crawl)" with a `title` explaining, `role="status"`.
4. Error state from `useDataset` → full-panel `EmptyState` with "Retry" (calls `reload`).

Commit: `stage-06: counts and empty states`.

---

## Acceptance criteria

- [ ] `pnpm -C web test` green; ≥ 60 tests total in the web package
- [ ] filters are fully keyboard-operable; every control has an accessible name
- [ ] no business logic in components (`FilterBar` only calls `onChange`)
- [ ] `pnpm -C web typecheck` clean, no `any`
- [ ] list of 200 generated buildings renders under the 300 ms budget
- [ ] links: `target="_blank"` + `rel="noopener noreferrer"` asserted in tests

## Result

| Task | Tests | Status |
|---|---|---|
| 6.1 | `FilterBar.test.tsx` (10) | |
| 6.2 | `App.test.tsx` (6) | |
| 6.3 | `BuildingRow/ResultsList.test.tsx` (7) | |
| 6.4 | `ResultsList.test.tsx` (4) | |

## Traps

- `type="number"` gives spinners and locale quirks — use text + `inputMode="numeric"`.
- Debounce **only** the building text input; size/beds/district must apply immediately or the UI
  feels broken.
- The row's clickable area must not swallow the unit link's click: `stopPropagation` on the link, or
  render units outside the `<button>`.
- Do not read `building.priceMin` from the dataset for display — after filtering it must be
  recomputed (`groupByBuilding`), otherwise a filtered row shows prices of hidden units.
