---
id: 05-12
stage: 05
title: Money, area and bed formatting
points: 1
owner: web
status: todo
depends_on: [05-03]
files:
  - web/src/lib/format.ts
  - web/src/lib/format.test.ts
---

# 05-12 · Money, area and bed formatting

**Points:** 1 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 05-03 · **Unblocks:** 06-07, 07-07

## Goal

Add `thb`, `sqm` and `beds` — the three display formatters used by every unit row and building panel — with hand-written grouping so the output is identical on every machine.

## Why this task exists

Prices, areas and bedroom counts appear in the unit row (06-07), the building panel (07-08) and the results rows, and if each component formats its own strings the same 45,000 becomes `฿45,000`, `45,000 THB` and `THB 45,000` in three places. Locale-aware formatting is the trap: `toLocaleString` emits a non-breaking space or a different symbol depending on the machine's ICU data, so a test would pass locally and the deployed page would differ. These helpers are pure, hand-grouped and asserted with exact strings, including the unit and sign edge cases.

## Current state (preconditions)

- 05-03 is done: `web/src/types.ts` exports the unit fields (`priceThb`, `sizeSqm`, `bedrooms`) and the fixture has 45,000 / 65 / 1-bed values to compare against.
- `web/src/lib/` contains `filter.ts`, `group.ts` and `viewport.ts`; there is no `format.ts`.
- `priceRange` and `countLabel` already exist in `group.ts` (05-10) — this task must not re-create or re-export them.

## Read before you start

- `CLAUDE.md` §3 (table-driven pure-function tests), §5 (one implementation per rule), §6, §7.
- `plan/02-architecture.md` §5.1 (row anatomy: `฿45,000`, `65 m²`, `1 bed`) and §5.3 (type scale, `tabular-nums`, no icon library).
- `plan/stage-05-web-shell.md` Task 5.3 — the `format.ts` contract and the sample expectations.
- `plan/90-test-matrix.md` row W-12 — the row that displays these strings; no dedicated matrix row exists for formatting.
- `web/src/lib/group.ts` — read `priceRange`/`countLabel` so the short labels and these formatters stay distinct.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `web/src/lib/format.ts` | `thb`, `sqm`, `beds`; no other exports |
| create | `web/src/lib/format.test.ts` | the six cases below |

Nothing else may change: no `group.ts` edit, no component, no hook, no dependency, no fixture edit.

## Test first (red)

Write exact-string tables — no `toMatch`, no `toContain` on the number — before the module exists.

1. `test_thb_groups_integer_baht_with_commas` — `thb(45000)` is `฿45,000`; `thb(0)` is `฿0`; `thb(1000)` is `฿1,000`; `thb(1000000)` is `฿1,000,000`; `thb(1234567)` is `฿1,234,567`. The symbol is U+0E3F immediately before the first digit.
2. `test_thb_puts_the_sign_before_the_symbol` — `thb(-1500)` is `-฿1,500` and `thb(-1)` is `-฿1`; sign first, no space, no parentheses.
3. `test_thb_rounds_fractional_baht_half_up` — `thb(45000.4)` is `฿45,000`; `thb(45000.5)` is `฿45,001`. Prices arrive as integers, but the formatter must not print a decimal point.
4. `test_sqm_formats_area_with_the_square_metre_sign` — `sqm(65)` is `65 m²`; `sqm(45)` is `45 m²`; `sqm(65.5)` is `65.5 m²`; `sqm(0)` is `0 m²`. The superscript is U+00B2 in the string `m²`.
5. `test_beds_labels_studio_and_plural` — `beds(0)` is `Studio`; `beds(1)` is `1 bed`; `beds(2)` is `2 beds`; `beds(3)` is `3 beds`; `beds(4)` is `4 beds`.
6. `test_output_is_locale_stable` — for `thb(1234567)`, `thb(-1500)`, `sqm(65.5)` and `beds(2)`: the strings contain no U+00A0 (NBSP) or U+202F (narrow NBSP), no space as a thousands separator, and no `Intl`-style currency code (`THB`) or decimal comma. Assert the exact strings too, so a machine with different ICU data fails loudly.

Expected red symptom: `Cannot find module './format'`. If a case fails with a type error after the module exists, fix the case before changing the module.

Run: `pnpm -C web test format` — six red for the missing module.

## Implement (green)

- `thb(n: number): string` — sign `-` for negative values (before the symbol), then `฿`, then the absolute value rounded half-up to an integer and grouped with a comma every three digits. Grouping is string arithmetic on the digits; no `Intl`, no `toLocaleString`, no `Number.toFixed` for grouping. No decimal part ever appears.
- `sqm(n: number): string` — the area, at most one decimal, with a trailing `.0` trimmed, then a single space and `m²` (U+00B2). Non-integers keep their one decimal; do not use `toLocaleString`.
- `beds(n: number): string` — `Studio` for 0, `1 bed` for 1, `${n} beds` otherwise. No leading zero, no space in `Studio`.
- One-sentence docstrings on all three, stating the exact output family. No `any`, no React import. Do not implement `priceRange`/`countLabel` here: those live in `group.ts` (05-10) and use a `k`/`M` short form, not the full baht string.
- Do not add a generic `formatNumber`/`formatCurrency` with options — three callers, three fixed contracts, no configuration.

Run: `pnpm -C web test format` — six green.

## Refactor / keep it DRY

One grouping routine, used by `thb` only; one decimal-trimming routine shared by `sqm`. Do not move `priceRange` here or re-export the group helpers — the short-range boundary table belongs with `priceRange` in `group.test.ts`, and duplicating it would give two formatters for one row. If a component needs a combined cell string, it composes these functions; it does not format regex-style.

## Verification (how to test it)

```text
pnpm -C web test format
pnpm -C web test --run
pnpm -C web typecheck
pnpm -C web build
make check
```

Expected: 6 passed for the target, 65 for the lane, typecheck 0, build clean, `make check` green. Manual: `grep -n 'toLocaleString\|Intl' web/src/lib/format.ts` prints nothing.

## Out of scope (do not do)

- No `priceRange`/`countLabel` (05-10), no currency conversion, no rounding mode options, no negative-area handling.
- No component, hook, or React import; no `Intl`/`toLocaleString`; no dependency.
- No thousands separators in `sqm`/`beds`, no unit tests of the layout or CSS.

## Definition of done

- [ ] All six cases exist in `web/src/lib/format.test.ts`, named as written, and pass.
- [ ] Red run observed and quoted; every assertion is an exact string, not a substring or a regex on the digits.
- [ ] `pnpm -C web typecheck` clean; lane suite and `make check` green; no `any`, no `Intl`.
- [ ] Only the two listed files are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled; exactly one commit on the trunk (`main`) with the message below.

## Commit

```text
05-12: add baht, area and bedroom formatters

- points: 1
- red: format.test.ts::test_thb_groups_integer_baht_with_commas failed (Cannot find module './format')
- add thb with hand-grouped integers and a sign before the symbol
- add sqm and beds with exact, locale-independent output
- suite: 65 passed
```

Files in the commit: `web/src/lib/format.ts`, `web/src/lib/format.test.ts`, and this task file.

## Traps and review notes

- `toLocaleString` is the trap: it yields a NBSP or a different symbol per ICU build, so the test must assert exact strings and the reviewer must grep for `Intl`.
- `m2` instead of `m²` (U+00B2) renders fine in a monospace diff and wrong on the page.
- Grouping by regex on the number's string form breaks for values below 1000 and for negative numbers; group the absolute digits first, then prepend the sign and symbol.
- `thb` must not print `.00`; prices are integers, and `45,000.00` breaks the row width and the design.
- Reviewers: `grep -n 'toLocaleString\|Intl\|toFixed' web/src/lib/format.ts` empty; `grep -n '−\|-฿' web/src/lib/format.test.ts` shows the sign test; `group.ts` untouched.

## Plan samples are untrusted

The plan shows expected strings (`฿45,000`, `65 m²`) but no code and no sign rule. Verify the exact code points (U+0E3F, U+00B2) and the installed TypeScript's handling of `Number.isInteger` before relying on them; do not paste a snippet (`CLAUDE.md` §6). If a plan sample disagrees on the sign placement or the area decimals, this task wins; record the deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Deviations | |
