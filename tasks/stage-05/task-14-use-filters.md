---
id: 05-14
stage: 05
title: Filter state hook with single-key updates and reset
points: 0.5
owner: web
status: todo
depends_on: [05-07]
files:
  - web/src/state/useFilters.ts
  - web/src/state/useFilters.test.ts
---
# 05-14 · Filter state hook with single-key updates and reset
**Points:** 0.5 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 05-07 · **Unblocks:** 05-17, 06-01
## Goal
Add `useFilters`, the one state holder for the user's filter values, updated one key at a time.
## Why this task exists
Filters are the app's main interaction: size range, bedrooms, district and a building-name search decide which
buildings are visible (rules F1–F5, `plan/02-architecture.md` §5.2). The rules themselves live in
`web/src/lib/filter.ts` (05-07…05-09); missing is the state the filter bar writes to and `App.tsx` reads. It
exists once, so the bar stays dumb, and unchanged values keep their identity so a keystroke that changes nothing
does not re-filter ~1000 buildings.
## Current state (preconditions)
- 05-07, 05-08, 05-09 done: `web/src/lib/filter.ts` exports `Filters` with five fields — `sizeMin`, `sizeMax`
  (`number | null`), `beds` (`"any" | "studio" | 1 | 2 | 3 | "4+"`), `district` (`string | "all"`), `building`
  (`string`) — plus `filterUnits`. Read the interface; import it, never restate it.
- 05-01/05-02 done: Vitest + jsdom + `@testing-library/react` wired, `pnpm -C web test --run` green.
  `web/src/state/` exists from 05-13; no `useFilters.ts` yet.
## Read before you start
- `CLAUDE.md` §3 (TDD, no network in unit tests), §5 (filters belong in `lib/filter.ts`; no `any` in
  `web/src/state/**`).
- `plan/stage-05-web-shell.md` Task 5.4 — the three members and the identity requirement.
- `web/src/lib/filter.ts` and its test — the field types and the "no filter" semantics already proven.
## Files to touch
| Action | Path | What goes there |
|---|---|---|
| create | `web/src/state/useFilters.ts` | the hook and its exported types |
| create | `web/src/state/useFilters.test.ts` | the four cases below |

Only these two files change; report anything else you believe is needed.
## Test first (red)
No rendering here: `renderHook` from the installed `@testing-library/react` is enough.

1. `useFilters.test.ts::test_initial_filters_match_the_no_filter_contract` — first render → `sizeMin` `null`,
   `sizeMax` `null`, `beds` `"any"`, `district` `"all"`, `building` `""`, and no sixth key.
2. `useFilters.test.ts::test_set_filter_changes_only_that_key` — `setFilter("sizeMin", 45)` → `45` with the other
   four unchanged; then `setFilter("building", "sathorn")` → `building` `"sathorn"`, `sizeMin` still `45`;
   `setFilter("sizeMax", null)` re-opens the upper bound.
3. `useFilters.test.ts::test_set_filter_with_the_same_value_keeps_the_object_identity` — call
   `setFilter("beds", "any")` while it already holds `"any"` → the returned `filters` is the **same reference**
   (`toBe`, not `toEqual`); same for `sizeMin` set to `null`.
4. `useFilters.test.ts::test_reset_returns_to_the_initial_filters` — set `sizeMin` 60, `beds` 2, `district`
   `"Khlong Toei"`, `building` `"premier"`, then `reset()` → all five back to the case-1 values; setters still work
   afterwards.

Expected red: `Failed to resolve import "./useFilters" from "src/state/useFilters.test.ts"` for every case; run
`pnpm -C web test --run useFilters` and copy one failure line.
## Implement (green)
Create `web/src/state/useFilters.ts` with one hook and two exported types:

- `useFilters()` → `{ filters, setFilter, reset }`;
  `setFilter<K extends keyof Filters>(key: K, value: Filters[K]): void`; `reset(): void`. `filters` is the
  imported `Filters` type.
- The initial value is one module constant with the five "no filter" values; `reset()` restores that same
  constant. `setFilter` returns the previous object unchanged when the value is strictly equal, otherwise a copy
  with only that key replaced (functional update).
- `setFilter` and `reset` keep stable identities across renders so `App.tsx` can memo on them. One-sentence
  docstring: it owns the values and applies no rules. No filtering, validation, clamping or debounce here.

Run: `pnpm -C web test --run useFilters` — four green.
## Refactor / keep it DRY
The defaults exist once, in the constant used by both the initial state and `reset()`. Do not restate the
`Filters` shape. If you need `filterUnits` inside the hook, the cut is wrong: the hook stores values, `App.tsx`
applies them.
## Verification (how to test it)
```text
pnpm -C web test --run useFilters
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: 4 passed, suite green, `tsc --noEmit` clean, `make check` green.
## Out of scope (do not do)
- Filter bar inputs, labels, debounce, validation — 06-01…06-04.
- Wiring into `App.tsx` — 05-17. Applying `filterUnits` here, or storing a filtered list as state.
- URL sync, `localStorage`, a reducer or state library (F10); per-field setters like `setBeds` (one generic setter
  is the contract).
## Definition of done
- [ ] All four cases exist, named as written, and pass.
- [ ] Red run observed and quoted in the commit body; target tests + lane suite + typecheck green.
- [ ] No `any`; `Filters` imported, not restated; only the two listed files plus this task file are modified.
- [ ] `status: done`, `## Result` filled; exactly one trunk commit with the message below.
## Commit
Message (first line exactly, then body):

```text
05-14: add filter state hook

- points: 0.5
- red: test_initial_filters_match_the_no_filter_contract failed (Failed to resolve import "./useFilters")
- add useFilters over the five-field Filters type, generic setFilter and reset
- unchanged values return the previous object so memos stay stable
- suite: <N> passed
```

Commit only the listed files plus this task file.
## Traps and review notes
- Always spreading into a new object, or comparing shallowly by deep equality: case 3 fails and every keystroke
  re-filters the dataset.
- Hardcoding `"any"`/`"all"` while `lib/filter.ts` defines them differently; import the types and read the "no
  filter" contract there.
- Storing filtered units or a `hasFilters` boolean here creates a second source of truth; both belong to
  `App.tsx`/`lib/`.
- Reviewers: grep `useFilters.ts` for `filter(`, `includes(`, `toLowerCase` — none may appear.
## Plan samples are untrusted
The signature in `plan/stage-05-web-shell.md` Task 5.4 is illustrative. Apply `CLAUDE.md` §6: read the real
`Filters` interface from 05-07…05-09 and the installed `@testing-library/react` types for `renderHook`/`act`. If
the repo disagrees with the sample, follow the repo and record it under `## Decisions`. Add no dependency.
## Decisions

## Result
| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
