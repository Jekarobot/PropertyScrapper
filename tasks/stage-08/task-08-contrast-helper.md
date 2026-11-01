---
id: 08-08
stage: 08
title: Contrast helper and accent token check
points: 0.5
owner: web
status: todo
depends_on: [05-12]
files:
  - web/src/lib/format.ts
  - web/src/lib/format.test.ts
---

# 08-08 · Contrast helper and accent token check

**Points:** 0.5 · **Stage:** 08 — Hardening and release · **Owner lane:** web · **Depends on:** 05-12 · **Unblocks:** 08-12 (product sign-off)

## Goal

Add a small WCAG contrast function to `lib/format.ts` and pin the two colour tokens the UI depends on — accent `#0f766e` and muted `#6b7280` on white — at or above 4.5:1.

## Why this task exists

The palette in `plan/02-architecture.md` §5.3 was chosen by eye: the accent is the selected bubble and link colour, the muted token carries district lines and hints, and both sit on white or `#fafafa`. Nobody has computed their contrast, so a
future "let's soften the teal" change would ship a page that is unusable in daylight with no test failing. A ten-line implementation of the WCAG relative-luminance formula, asserted against the definitional pairs (black/white is 21, a colour
against itself is 1) and against the two tokens, turns a taste decision into a check. It lands in `format.ts` beside the other display helpers: no dependency, no fixture, no browser.

## Current state (preconditions)

- 05-12 is done: `web/src/lib/format.ts` exports `thb`, `sqm` and `beds`, each with a one-sentence docstring, and `format.test.ts` has six green cases.
- The palette lives in `plan/02-architecture.md` §5.3 and in Tailwind classes across components; there is no token module and this task must not create one.
- `pnpm -C web test --run` and `pnpm -C web typecheck` are green; the module has no `any`.
- The function has no product caller yet: it exists to be asserted, and later to be quoted in the runbook.

## Read before you start

- `CLAUDE.md` §3 (table-driven pure-function tests, no network), §5 (no dependency, no abstraction with one implementation), §6, §7.
- `plan/02-architecture.md` §5.3 — the exact hex values: accent `#0f766e`, muted `#6b7280`, text `#111827`, background `#fafafa`, surface `#ffffff`.
- `plan/stage-08-hardening-ops.md` Task 8.4 item 4 — the accent threshold and the "10 lines, no dependency" constraint.
- `plan/90-test-matrix.md` row W-17; `web/src/lib/format.ts` and `format.test.ts` — the existing style and exports.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/lib/format.ts` | `contrastRatio` and nothing else |
| modify | `web/src/lib/format.test.ts` | the six cases below |

Nothing else: no new module, no token export, no component, no dependency, no CSS change.

## Test first (red)

Exact numbers where the definition fixes them, thresholds where the design fixes them, and no hardcoded ratio for the two tokens — the test computes.

1. `test_contrast_ratio_black_on_white_is_21` — `contrastRatio("#000000", "#ffffff")` is 21 within five decimals (the formula's maximum, `1.05 / 0.05`), and the reversed arguments give the same value. Catches a missing gamma correction or a
   swapped luminance order.
2. `test_contrast_ratio_of_a_colour_with_itself_is_1` — `("#0f766e", "#0f766e")` and `("#ffffff", "#ffffff")` are exactly 1.
3. `test_accent_token_clears_4_5_on_white` — `contrastRatio("#0f766e", "#ffffff") >= 4.5`. The accent colours links and the selected bubble; 4.5:1 is the WCAG AA body-text floor.
4. `test_muted_token_clears_4_5_on_white` — `contrastRatio("#6b7280", "#ffffff") >= 4.5`. Compute it in the test; if the shipped token is below the floor, stop and report to the owner rather than lowering the threshold or editing the palette
   here.
5. `test_contrast_ratio_is_bounded_and_monotonic` — an intermediate grey (`#777777`) on white gives a value strictly between 1 and 21 and strictly between the results for `#000000` and `#ffffff`; `#fafafa` on white is just above 1. This is what
   proves the function is not a lookup table returning the two definitional answers.
6. `test_three_digit_hex_and_garbage_are_rejected` — `contrastRatio("#fff", "#ffffff")` and `contrastRatio("teal", "#ffffff")` throw an `Error` whose message contains the offending input. No silent shorthand expansion, no NaN: a typo in a future
   colour must fail loudly.

Expected red: `contrastRatio is not a function` (or a TypeScript error for the missing export) on case 1. Run `pnpm -C web test --run format` and quote the line.

## Implement (green)

- `contrastRatio(foreground: string, background: string): number` in `web/src/lib/format.ts`, exported beside the formatters: parse `#rrggbb`, convert each channel to its linear sRGB value, compute `0.2126 R + 0.7152 G + 0.0722 B`, then
  `(lighter + 0.05) / (darker + 0.05)`. Round only at the end (two decimals is enough) and never clamp a legitimate result.
- Order-independent: the same pair in either argument order returns the same number.
- Reject anything that is not exactly seven characters starting with `#` followed by six hex digits, with an `Error` naming the input. No "warn if identical" branch.
- No token constants, no CSS parsing, no computed styles, no colour library: callers pass literals, and the stylesheet stays the single place a colour is defined for rendering.
- One-sentence docstring naming the formula, the input form and the range (1–21).

Run: `pnpm -C web test --run format` — twelve green.

## Refactor / keep it DRY

One luminance helper and one ratio function, both private except the exported ratio; the existing formatters stay untouched. The two hex values appear in this test file only as assertions on the palette, not as a second token registry — do not
export them "so components can use them", because the components already use Tailwind classes.

## Verification (how to test it)

```text
pnpm -C web test --run format
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: twelve passed in the target file; the suite and typecheck green; `make check` exits 0. No visual change is expected — if the diff touches a component, something was done in the wrong file.

## Out of scope (do not do)

- A token module, CSS custom properties, a Tailwind palette edit, or re-theming any component.
- Checking hover, disabled or `#14b8a6` (the idle bubble at 80 % opacity); one accent pair is the contract.
- A `colors`, `culori` or `axe` dependency, or a snapshot of computed styles.
- Running a browser or a screenshot; this is a pure function.

## Definition of done

- [ ] All six cases exist, named as written, and pass.
- [ ] Red run observed and quoted in the commit body.
- [ ] The two token ratios are computed in the test, not pasted as decimals in the source.
- [ ] `pnpm -C web test --run` and `pnpm -C web typecheck` green; `make check` exits 0.
- [ ] Only the two listed files and this task file changed (`git status --short`).
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and the two measured ratios.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
08-08: add a WCAG contrast helper and pin the palette

- points: 0.5
- red: test_contrast_ratio_black_on_white_is_21 failed (contrastRatio is not a function)
- add contrastRatio with the WCAG relative-luminance formula, no dependency
- assert black/white 21, same colour 1, and both palette tokens at or above 4.5:1
- suite: 47 web tests
```

Files in the commit: `web/src/lib/format.ts`, `web/src/lib/format.test.ts`, this task file. One commit, on trunk `main`.

## Traps and review notes

- The common wrong formula skips the sRGB transfer function (`c / 12.92` below 0.04045) or uses 255 instead of 1 as the channel maximum; both still make black/white pass, so case 5's monotonicity assertion is the one that catches them.
- Hardcoding 4.74 as the expected muted ratio makes the test a second source of truth: assert the threshold against the hex and paste the measured number in `Result`.
- A helper that accepts 3-digit shorthand silently changes the meaning of `#fff` versus `#ffffff` and hides typos; case 6 exists for that.
- Rounding inside the luminance computation shifts values near 4.5; round only the final ratio.
- Reviewers: `grep -n '0.2126' web/src/lib/format.ts` is one line; `grep -rn 'contrastRatio' web/src` finds the module and its test only; no new entry in `package.json`.

## Plan samples are untrusted

The plan says "a tiny contrast helper in `lib/format.ts` — 10 lines, no dependency" and gives no formula or input contract. The WCAG 2.1 definition is the authority; verify the constants against it and record any deviation (input form, rounding,
error type) under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |

