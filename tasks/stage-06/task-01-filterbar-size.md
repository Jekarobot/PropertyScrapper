---
id: 06-01
stage: 06
title: Filter bar size inputs with debounce and validation
points: 2
owner: web
status: todo
depends_on: [05-14, 05-19]
files:
  - web/src/components/FilterBar.tsx
  - web/src/components/FilterBar.test.tsx
---

# 06-01 · Filter bar size inputs with debounce and validation
**Points:** 2 · **Stage:** 06 — Filters and results list · **Owner lane:** web · **Depends on:** 05-14, 05-19 · **Unblocks:** 06-02

## Goal
Create `FilterBar` with its full props contract and the size min/max pair: text inputs with numeric input mode, 200 ms debounce, empty = "no bound", non-numeric rejected, invalid range flagged.

## Why this task exists
The filter bar is the only way a visitor narrows the list; without it the app is one fixed view of the whole dataset. The size pair is the hardest control — it debounces, maps empty to `null`, must never produce `NaN`, and has a cross-field
rule — so it comes first and fixes the component's shape for the rest: controlled, dumb, reporting every change through one callback and filtering nothing (`CLAUDE.md` §5). 06-02 to 06-04 fill the other slots without changing the interface;
06-05 wires the bar into `App`.

## Current state (preconditions)
- 05-14 done: `web/src/state/useFilters.ts` exports `Filters` and the hook, defaults `null`, `null`, `"any"`, `"all"`, `""`.
- 05-19 done: `App.tsx` renders the 64 px filter-bar slot with disabled placeholder inputs; status bar present.
- 05-03/05-12 done: `web/src/test/fixtures/dataset.sample.json` (6 units / 3 buildings) and `web/src/lib/format.ts` exist.
- `web/src/components/FilterBar.tsx` and its test do **not** exist.

## Read before you start
- `CLAUDE.md` §3, §5 (filters live in `lib/filter.ts`; components stay dumb), §6, §7.
- `plan/stage-06-filters.md` Task 6.1 — props, items 1–4 and 9–10, Traps (`type="number"`).
- `plan/02-architecture.md` §5.1 (bar anatomy), §5.2 F1/F4 (inclusive bounds, empty = unbounded, invalid ignored, never `NaN`), §5.3 (border `#e5e7eb`, muted `#6b7280`, accent `#0f766e`, radius 10).
- `plan/90-test-matrix.md` row W-11; `web/src/state/useFilters.ts` for the exact `Filters` shape.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `web/src/components/FilterBar.tsx` | props interface, size pair, debounce, size parsing |
| create | `web/src/components/FilterBar.test.tsx` | the seven cases below plus one render helper |

Nothing else may change: the bar is not mounted in `App.tsx` here (06-05) and no other control is built.

## Test first (red)
Write literals read from the fixture; never derive an expectation by calling `filterUnits` inside the test.

1. `test_size_inputs_have_accessible_names` — `getByLabelText("Size minimum")` and `"Size maximum"` resolve; each is an `input` with `type="text"` and `inputMode="numeric"` (not `number`); minimum precedes maximum in document order.
2. `test_inputs_show_the_current_filter_values` — `sizeMin: 60`, `sizeMax: 120` display `60` and `120`; both `null` display empty strings, never the word `null`.
3. `test_typing_minimum_calls_on_change_once_after_the_debounce` — fake timers, type `60`: no call before 200 ms, exactly one `("sizeMin", 60)` after; configure user-event to advance the fake timers.
4. `test_clearing_a_size_field_reports_null_immediately` — clearing the prefilled maximum reports `("sizeMax", null)` without waiting for the debounce.
5. `test_non_numeric_input_is_rejected_and_keeps_the_last_valid_value` — after committing `60`, typing `abc` (also `" "` and `"6O"`) leaves `60` visible and produces no further `onChange`.
6. `test_max_below_min_marks_the_field_invalid_and_shows_an_alert` — `sizeMin` `100`, type `50`: the maximum gets `aria-invalid="true"`, is described by a `role="alert"` reading exactly `Max size is below min size`, and `("sizeMax", 50)` still
   fires.
7. `test_bar_is_not_a_form_and_enter_does_not_submit` — the root is not a `form`; Enter in the maximum field produces no submit and no extra `onChange`.

Expected red: `Unable to find a label with the text of: Size minimum`, or the unresolved `./FilterBar` import. A syntax error is broken, not red. Run `pnpm -C web test --run FilterBar`, copy one failure line.

## Implement (green)
Export `FilterBar` and `FilterBarProps` with exactly `filters`, `districts`, `buildingOptions`, `resultCount`, `onChange`, `onReset`. Declare the interface whole here because 06-05 wires it; later tasks fill slots and must not change it.
Destructure only what you render.

- Root: `section` named `Filters` (or `fieldset` + visually hidden legend; 06-04 pins the group name). No `form`.
- Accessible names exactly `Size minimum` / `Size maximum`, via `aria-label` or hidden `label` text; the visible compact line may read `Size [min]–[max] m²`.
- Local draft text for both fields, following `filters` when it changes from outside (reset): the component is controlled by `filters`, drafts only echo it.
- Parse: trim, then accept only digits (optionally one `.` with digits on both sides). Empty → `null`; anything else → rejected, keep the last valid draft, no `onChange`. Never emit `NaN` or a string.
- One 200 ms timer shared by the pair (one helper in this file; 06-03 reuses it): debounced commit sends the number, a clear sends `null` immediately and cancels the timer.
- Validation compares drafts when both parse: maximum strictly below minimum → `aria-invalid="true"`, `aria-describedby` to a `role="alert"` reading `Max size is below min size`, present only while invalid. Never block `onChange`; the filter
  layer decides.
- No `filterUnits`, dataset access, formatting or lowercasing. One comment: "controlled filter bar; reports changes, never filters".

Run `pnpm -C web test --run FilterBar` — seven green.

## Refactor / keep it DRY
One parser and one debounce helper in this file, reused by 06-03/06-04. Do not import `lib/filter.ts`. Do not add a shared hook — that path is not in this task; a third caller means report, not refactor.

## Verification (how to test it)
```text
pnpm -C web test --run FilterBar
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: 7 passed; whole web suite green; `tsc --noEmit` clean, no `any`; `make check` exits 0.

## Out of scope (do not do)
- Beds/district selects (06-02), building input and datalist (06-03), Reset, count, fieldset and tab order (06-04), mounting in `App.tsx` (06-05), filtering or URL sync, and any map work (stage 07).

## Definition of done
- [ ] Seven cases exist, named as written, and pass.
- [ ] Red run observed and quoted in the commit body.
- [ ] Target tests, web suite, typecheck green; `make check` exits 0.
- [ ] `git status --short` lists only the two files above.
- [ ] `status: done`; `## Result` filled with hash and counts; exactly one commit on `main`.

## Commit
```text
06-01: add filter bar size inputs with debounce

- points: 2
- red: test_size_inputs_have_accessible_names failed (Unable to find a label with the text of: Size minimum)
- add FilterBar props contract and the size min/max pair
- debounce 200 ms, empty maps to null, non-numeric input rejected
- invalid range sets aria-invalid and a role=alert hint without blocking onChange
- suite: 7 passed
```

Commit this task alone, with the listed files only, on the trunk (`main`) — one task = one
commit, no branch, no push.

## Traps and review notes
- `type="number"` is banned: spinners, locale quirks, and an empty value that is not `null`.
- Debounce only the size pair (and later the building text); clearing must be immediate or the range cannot be widened quickly.
- Rejecting input must not mutate the committed filter, and the draft keeps the last valid value rather than the rejected text.
- Compare drafts, not committed values, or the alert appears only after the debounce and then flickers.
- No lowercasing here: F4 matching lives in `lib/filter.ts`.
- Reviewer greps `type="number"`, `NaN`, `parseInt`, `form`, `lib/filter` in `FilterBar.tsx`: all absent.

## Plan samples are untrusted
The stage plan shows a props sketch, not component code. Check any borrowed API (user-event fake-timer options, React version) against `web/package.json` and the installed typings before trusting it (`CLAUDE.md` §6); record deviations under
`## Decisions`.

## Decisions
## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
