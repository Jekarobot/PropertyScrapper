---
id: 08-10
stage: 08
title: Reduced-motion support
points: 0.5
owner: web
status: todo
depends_on: [07-12]
files:
  - web/src/index.css
  - web/src/components/BuildingPanel.tsx
  - web/src/components/BuildingPanel.test.tsx
  - web/src/test/media.ts
---

# 08-10 · Reduced-motion support

**Points:** 0.5 · **Stage:** 08 — Hardening and release · **Owner lane:** web · **Depends on:** 07-12 · **Unblocks:** 08-12 (product sign-off)

## Goal

Honour `prefers-reduced-motion: reduce`: one CSS media block that turns off the Leaflet tooltip and popup fades, and a panel that drops its own transition when the setting is on.

## Why this task exists

The panel slides in on selection and Leaflet fades its tooltips; for a user who set "reduce motion" in the OS — a real setting for vestibular disorders, not a preference toggle — those are the animations that make a map product unpleasant. There
are two mechanisms because there are two owners: our panel is a component with a class, and the tooltip is Leaflet's own CSS, which we cannot edit in `node_modules`. jsdom cannot evaluate media queries, so the component reads `matchMedia` and
the CSS block is asserted as text; that split is deliberate and the last case keeps the two from drifting apart. 07-12 added the mobile sheet with its own transition, which is why this lands after it.

## Current state (preconditions)

- 05-01/05-02 are done: `web/src/index.css` is the single global stylesheet loaded by `main.tsx`, with the Tailwind directives and the shell rules.
- 07-07/07-08 are done: `BuildingPanel.tsx` renders the left panel, keeps the placeholder when nothing is selected, and carries the slide transition class.
- 07-12 is done: the mobile sheet has its own open/close transition and reuses the panel styles.
- `web/src/components/BuildingPanel.test.tsx` exists and is green; `pnpm -C web typecheck` is clean; no animation library is installed.

## Read before you start

- `CLAUDE.md` §3 (jsdom has no media queries — assert the CSS as text, stub `matchMedia`), §5 (no dependency), §6, §7.
- `plan/stage-08-hardening-ops.md` Task 8.4 item 6 — "@media (prefers-reduced-motion: reduce) disables the tooltip fade".
- `plan/02-architecture.md` §5.3 — the animation budget ("no gradients, subtle shadow") and §5.2 F9 for when the panel appears.
- `plan/90-test-matrix.md` row W-17; `web/src/index.css`; `web/src/components/BuildingPanel.tsx`; the installed `leaflet/dist/leaflet.css` for the tooltip and popup selectors.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/index.css` | one reduced-motion block for the third-party surfaces |
| modify | `web/src/components/BuildingPanel.tsx` | the matchMedia read and the transition class choice |
| modify | `web/src/components/BuildingPanel.test.tsx` | the four cases below |
| modify | `web/src/test/media.ts` | the one fake answers the motion query |

Nothing else: no new module, no second `matchMedia` mock, no `tailwind.config` change, no dependency, no e2e.

## Test first (red)

The file already has a render helper and a props fixture for the panel. Extend the single fake in `web/src/test/media.ts` (07-12) with a reduced-motion switch: one exported setter beside `setViewportWidth`, a `motionQuery()` reader, and a record of the last query asked. Do not install a per-test `window.matchMedia = vi.fn()` — 07-12's trap list forbids the second fake.

1. `BuildingPanel.test.tsx::test_index_css_declares_a_reduced_motion_block_for_third_party_surfaces` — read `web/src/index.css` as raw text and assert it holds exactly one `@media (prefers-reduced-motion: reduce)` block whose body names the
   Leaflet tooltip and popup selectors and the panel's transition property. Exactly one: a second block added later is how the two mechanisms drift.
2. `::test_panel_drops_its_transition_when_reduced_motion_is_preferred` — reduced motion switched on in the fake, render with a selected building → the panel root does not carry the transition utility and exposes `data-reduced-motion="true"`; the name, content and unit count render exactly as before.
3. `::test_panel_keeps_its_transition_when_motion_is_allowed` — motion allowed → the transition utility is present and `data-reduced-motion` is absent (or `"false"`); no other class differs.
4. `::test_the_component_asks_about_the_same_query_the_css_declares` — the fake recorded exactly `(prefers-reduced-motion: reduce)`, equal to the query in the CSS block from case 1. A typo on either side makes the feature silently inert while both tests still look reasonable.

Expected red: `TypeError: window.matchMedia is not a function` inside the panel, and `data-reduced-motion` absent on case 2. Run `pnpm -C web test --run BuildingPanel` and quote the line.

## Implement (green)

- `index.css`: one block at the end of the file, `@media (prefers-reduced-motion: reduce)`, containing the Leaflet overrides — the tooltip and popup fade/zoom transitions set to none — plus the shared panel/sheet transition property. Use the
  class names the installed Leaflet ships (`.leaflet-tooltip`, `.leaflet-popup` and its fade classes), verified in `node_modules`.
- `BuildingPanel.tsx`: read the preference once when the panel mounts (`window.matchMedia` with the literal query, guarded for environments without it), store a boolean, choose between the transition utility and nothing, and expose the decision
  as `data-reduced-motion`. Do not subscribe to changes: the OS setting rarely changes mid-session and a listener is state this component does not need.
- The animated surface set stays as it is: nothing else animates, so nothing else needs a rule. Do not add a global `* { transition: none }` reset — it would also kill the focus ring's transition and make 08-07's ring feel broken.
- No JS animation, no `requestAnimationFrame`, no library, no post-mount effect to set the class (the first render already knows the answer). One-sentence comment above the media block saying it serves the third-party surfaces and that our
  components decide in JS.

Run: `pnpm -C web test --run BuildingPanel` — the four new cases green.

## Refactor / keep it DRY

One literal for the media query, in the component, with a test pinning it against the CSS; one transition class name referenced from the component and the CSS block. Do not add a `useReducedMotion` hook for a single consumer, and do not
duplicate the Leaflet override into a component style attribute.

## Verification (how to test it)

```text
pnpm -C web test --run BuildingPanel
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: four new cases pass; suite and typecheck green; `make check` exits 0. Manual check with devtools' "Emulate CSS prefers-reduced-motion": selecting a building shows the panel without sliding and a tooltip without fading; with the
emulation off both animate as before.

## Out of scope (do not do)

- A settings toggle in the UI, a stored preference, a `matchMedia` change subscription, or a `data-*` attribute on every component.
- Animating anything new, a duration scale in `tailwind.config`, a keyframes block, or a third-party animation package.
- Changing the tooltip or popup content, the map behaviour, or the focus ring's transition.
- A Playwright test for motion (the e2e suite asserts layout, not timing).

## Definition of done

- [ ] All four cases exist, named as written, and pass.
- [ ] Red run observed and quoted in the commit body.
- [ ] Exactly one reduced-motion block in `index.css`, naming the Leaflet selectors and the panel.
- [ ] `pnpm -C web test --run` and `pnpm -C web typecheck` green; `make check` exits 0.
- [ ] Only the three listed files and this task file changed (`git status --short`); no dependency.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and the test count.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
08-10: honour prefers-reduced-motion

- points: 0.5
- red: test_panel_drops_its_transition_when_reduced_motion_is_preferred failed (matchMedia is not a function)
- add one media block for the Leaflet tooltip and popup surfaces
- make the panel choose its transition from matchMedia; pin the query in a drift test
- suite: 51 web tests
```

Files in the commit: `web/src/index.css`, `web/src/components/BuildingPanel.tsx`, `web/src/components/BuildingPanel.test.tsx`, this task file. One commit, on trunk `main`.

## Traps and review notes

- The failure mode is silence: a typo in either the CSS query or the component's string leaves both tests plausible while the animation keeps running. Case 4 exists only for that.
- `.leaflet-tooltip` is not the class carrying the fade on every Leaflet version; check the installed stylesheet for the fade classes and override those too, with the version you verified.
- A global `* { transition: none !important }` passes the visual check and breaks the focus-ring transition from 08-07; scope the block to the named surfaces.
- Reading `index.css` from a test is a path dependency: use the project alias or a relative import that works from the test file, and assert the file is non-empty first so a moved file cannot make the assertions vacuous.
- Reviewers: `grep -c 'prefers-reduced-motion' web/src/index.css` is 1; the string appears in the CSS, the component and the test only; no `animate-*` class was added.

## Plan samples are untrusted

The plan says only "@media (prefers-reduced-motion: reduce) disables the tooltip fade" — no selector, no panel rule and no mechanism for the component side. Verify the Leaflet class names in the installed `leaflet.css` before writing the
override, and record the chosen mechanism (CSS for third-party, JS for our component) under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |

