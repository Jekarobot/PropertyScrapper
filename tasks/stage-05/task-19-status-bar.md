---
id: 05-19
stage: 05
title: Status bar with local freshness time and partial-data warning
points: 1
owner: web
status: todo
depends_on: [05-18]
files:
  - web/src/components/StatusBar.tsx
  - web/src/components/StatusBar.test.tsx
  - web/src/App.tsx
  - web/src/App.test.tsx
---
# 05-19 · Status bar with local freshness time and partial-data warning
**Points:** 1 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 05-18 · **Unblocks:** 06-01
## Goal
Add the real `StatusBar` — freshness time in the browser locale, a warning when the dataset is partial, and a
tooltip with crawl details — and mount it in the header.
## Why this task exists
Rule F11 (`plan/02-architecture.md` §5.2) requires the header to show `generatedAt` in local time and to warn
when `stats.partial` is true; §3.3 makes `partial` the honest signal that a crawl was aborted or a page failed.
"How old is this number?" and "is it complete?" are the first questions a stale or short dataset provokes, and
both answers belong in one component so the header never grows a second freshness line. Values come from the
dataset, never the browser clock, so the same file renders the same text.
## Current state (preconditions)
- 05-18 done: `App.tsx` has a `header` slot for the status bar; `App.test.tsx` has ten green cases.
- 05-03 done: `web/src/types.ts` has `Stats`; the sample fixture's `stats` has `units`, `buildings`,
  `buildingsWithCoords`, `pagesScanned`, `duplicatesDropped`, `excluded`, `partial`. `web/src/components/` may not
  exist yet — create it here. No icon library is installed and none may be added.
## Read before you start
- `CLAUDE.md` §3 (determinism: never read the clock in the component), §5 (YAGNI — no tooltip or icon package).
- `plan/02-architecture.md` §3.3 — the `stats` fields and the meaning of `partial`; §5.1 — the header line; §5.2
  rule F11; §5.3 — "no icon library beyond 4 inline SVGs".
- `plan/stage-05-web-shell.md` Task 5.5 — "StatusBar is real logic (not a placeholder)"; `plan/90-test-matrix.md`
  row W-10.
## Files to touch
| Action | Path | What goes there |
|---|---|---|
| create | `web/src/components/StatusBar.tsx` | the component and its props type |
| create | `web/src/components/StatusBar.test.tsx` | the four cases below |
| modify | `web/src/App.tsx` | mount it in the header |
| modify | `web/src/App.test.tsx` | the wiring case below |

Only these four files change; report anything else you believe is needed.
## Test first (red)
Component tests pass props directly — no fetch, no `App`.

1. `StatusBar.test.tsx::test_freshness_uses_intl_datetimeformat_for_the_frozen_timestamp` —
   `generatedAt: "2026-11-01T21:45:00Z"`, complete stats → the text equals `Updated ` plus
   `new Intl.DateTimeFormat(undefined, { day: "2-digit", month: "2-digit", hour: "2-digit", minute: "2-digit" }).format(new Date("2026-11-01T21:45:00Z"))`,
   computed in the test as the oracle; it must not contain the raw ISO string.
2. `StatusBar.test.tsx::test_partial_stats_render_a_warning_with_role_status` — `stats.partial: true` → exactly one
   `role="status"` element whose text contains `Partial data` and which holds an `svg` marked `aria-hidden="true"`
   (the icon adds no second accessible name).
3. `StatusBar.test.tsx::test_complete_stats_render_no_warning` — `stats.partial: false` → no `role="status"`
   element, freshness still rendered.
4. `StatusBar.test.tsx::test_tooltip_reports_pages_scanned_and_buildings_with_coords` — `pagesScanned: 73`,
   `buildingsWithCoords: 940`, `buildings: 987` (the documented §3.3 example values) → the element carrying the
   tooltip has a `title` containing both `73` and `940`.
5. `App.test.tsx::test_header_shows_freshness_from_the_loaded_dataset` — render `<App />` with the stubbed sample →
   the header contains `Updated` and the formatted value of the fixture's own `generatedAt`, and no partial warning
   (the sample is complete).

Expected red: `Failed to resolve import "./StatusBar" from "src/components/StatusBar.test.tsx"` for cases 1–4
and an assertion failure for case 5. Run `pnpm -C web test --run StatusBar` then `--run App`; copy one failure
line.
## Implement (green)
Create `web/src/components/StatusBar.tsx`:

- Props `{ generatedAt: string; stats: Stats }`, with `Stats` imported from `web/src/types.ts`; no `any`, no
  optional alternates.
- Freshness: one `Intl.DateTimeFormat` with the browser locale and two-digit day, month, hour and minute (exact
  options in case 1), applied to `new Date(generatedAt)` and rendered as `Updated <formatted>`; build it once,
  never read `Date.now()`.
- Warning: only when `stats.partial` — a `role="status"` element with an inline `svg` (`aria-hidden="true"`) and
  the text `Partial data`.
- Tooltip: a `title` on the bar's root stating pages scanned and buildings with coordinates, e.g.
  `Built from 73 pages; 940 of 987 buildings have coordinates`.
- Styling: header-scale text only, muted token; no button, popover or icon library. One-sentence docstring on the
  component.

Then mount it in `App.tsx`'s header slot with `data.generatedAt` and `data.stats`; do not format dates in
`App.tsx`.

Run: `pnpm -C web test --run StatusBar` and `pnpm -C web test --run App` — all green.
## Refactor / keep it DRY
Formatting lives only here: no other component may read `generatedAt`, and `App.tsx` must not format dates.
## Verification (how to test it)
```text
pnpm -C web test --run StatusBar
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: 5 new cases pass (4 component, 1 App), suite green, `tsc --noEmit` clean, `make check` green; manual:
`pnpm -C web dev` shows `Updated <local date>` in the header.
## Out of scope (do not do)
- A refresh button, polling or a "stale" threshold — nothing here re-fetches on a timer.
- A tooltip/popover library or an icon package: `title` and one inline SVG are the contract.
- Timezone selection, 12/24-hour settings, or reading `generatedAt` anywhere but this component.
## Definition of done
- [ ] All five cases exist, named as written, and pass.
- [ ] Red run observed and quoted; target tests + lane suite + typecheck + `make check` green.
- [ ] No `any`, no `Date.now()`, no new dependency; one inline SVG is the only icon.
- [ ] Only the four listed files plus this task file are modified; `status: done`, `## Result` filled; one trunk
      commit.
## Commit
Message (first line exactly, then body):
```text
05-19: add status bar with freshness and partial warning

- points: 1
- red: test_freshness_uses_intl_datetimeformat_for_the_frozen_timestamp failed (Failed to resolve import "./StatusBar")
- add StatusBar: Intl.DateTimeFormat freshness, role=status partial warning, stats tooltip
- mount it in the App header from the dataset's own generatedAt and stats
- suite: <N> passed
```

Commit only the listed files plus this task file.
## Traps and review notes
- Formatting in `App.tsx` as well gives two outputs that drift; one formatter, one place.
- Hardcoding an English month or fixed date makes the test pass and the product wrong for other locales.
- Reviewers: grep `web/src` for `toLocaleString`, `getFullYear`, `Date.now` (none) and for
  `new Intl.DateTimeFormat` (exactly one call site).
## Plan samples are untrusted
Task 5.5 gives requirements, not a signature, and the §5.1 header sketch shows `01.11 21:45` — one locale's
rendering, not a format string. Apply `CLAUDE.md` §6: read the installed React and Tailwind versions, confirm
the locale behaviour with the oracle above, and record deviations under `## Decisions`. Add no dependency.
## Decisions

## Result
| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
