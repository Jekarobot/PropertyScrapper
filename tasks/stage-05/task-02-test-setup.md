---
id: 05-02
stage: 05
title: Test setup, jsdom stub, smoke render
points: 0.5
owner: web
status: todo
depends_on: [05-01]
files:
  - web/src/test/setup.ts
  - web/src/test/setup.test.tsx
---

# 05-02 · Test setup, jsdom stub, smoke render

**Points:** 0.5 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 05-01 · **Unblocks:** 05-04, 05-07…05-12, 05-13

## Goal

Finish the shared Vitest setup — jest-dom matchers plus a ResizeObserver stub — and prove it with a smoke test that renders, asserts and clicks in jsdom.

## Why this task exists

From 05-04 onward every task mounts something: the loader tests need a DOM, the pure-library tasks need `describe`/`expect`, and the map tasks in stage 07 mount Leaflet, which constructs `ResizeObserver` at import time — jsdom does not provide one, so without the stub a Leaflet import crashes the whole suite. A broken setup fails dozens of later tests for a reason unrelated to the code under test, so it is proven once, here, with a rendered component and a real user-event click.

## Current state (preconditions)

- 05-01 is done: `web/vite.config.ts` has the `test` block with jsdom, `globals: true` and `setupFiles` → `src/test/setup.ts`; `web/src/test/setup.ts` exists but contains only the jest-dom/vitest import.
- `@testing-library/react`, `@testing-library/user-event`, `@testing-library/jest-dom` and `jsdom` are installed and pinned; `web/src/test/scaffold.test.ts` (7 cases) is green.
- `pnpm -C web test --run` currently exits 0.

## Read before you start

- `CLAUDE.md` §3 (no skipped tests, behaviour-named tests), §5 (no `any`), §6 (plan samples untrusted).
- `plan/stage-05-web-shell.md` Task 5.1 step 6 — the stub requirement and why it is for Leaflet.
- `plan/02-architecture.md` §4, the "Leaflet in jsdom" paragraph — component tests mock `react-leaflet`; the real library is only exercised in Playwright (stage 07).
- `plan/90-test-matrix.md` rows W-01…W-06, W-14 — these are the suites the setup must serve.
- `web/vite.config.ts` and `web/src/test/setup.ts` — read what 05-01 actually left.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/test/setup.ts` | jest-dom import (kept) + the ResizeObserver stub |
| create | `web/src/test/setup.test.tsx` | the three smoke cases below |

Nothing else may change: no `vite.config.ts` edit, no per-test helper file.

## Test first (red)

Write `web/src/test/setup.test.tsx` before touching `setup.ts`. Each case builds its own tiny component inside the test file; no product component is imported.

1. `test_renders_a_component_and_uses_jest_dom_matchers` — render a local component returning a heading whose text is `Connex Watch`; `screen.getByRole("heading", { name: "Connex Watch" })` exists and `toBeInTheDocument()` passes; `document.body` is the jsdom body. Proves the jsdom environment, RTL and the matcher registration together.
2. `test_resize_observer_stub_exists_and_is_a_noop` — `typeof globalThis.ResizeObserver === "function"`; constructing one with a callback, then `observe(document.body)`, `unobserve(document.body)` and `disconnect()` all return without throwing; a second instance can be constructed independently. This is the API Leaflet calls in stage 07.
3. `test_user_event_click_updates_rendered_text` — a local button with a counter; `await userEvent.click(button)` twice, then the rendered text is `2`. Proves user-event's async act wrapper works under `globals: true`.

Expected red symptom: `ReferenceError: ResizeObserver is not defined` at case 2 (and `TypeError: ResizeObserver is not a constructor` if a stub exists but is not one). Cases 1 and 3 may already pass once 05-01's setup runs; write all three anyway and quote case 2's line.

Run: `pnpm -C web test setup` — case 2 red, cases 1 and 3 green.

## Implement (green)

`web/src/test/setup.ts`, in this order:

- Keep the existing `@testing-library/jest-dom/vitest` import — it registers the matchers on Vitest's `expect` and augments the types.
- If `globalThis.ResizeObserver` is absent, define it as a class whose constructor accepts the observer callback and whose `observe`, `unobserve` and `disconnect` methods are no-ops. Install it as a real global constructor; do not replace it with a bare object or a mock function. One sentence above it says it exists so Leaflet can construct itself under jsdom.
- Do not export anything from `setup.ts`; it is loaded by `setupFiles`, not imported by tests.

Do not weaken the tests to fit the environment: if case 1 cannot find the heading, the setup file is not loading, not the test.

Run: `pnpm -C web test setup` — three green. Then the whole suite: ten green.

## Refactor / keep it DRY

The stub exists once, globally, in `setup.ts`; no test may define its own `ResizeObserver`. No test imports `setup.ts` directly. Delete any duplicated jest-dom import a test file might have added while debugging. RTL auto-cleanup already runs because `globals: true` gives `afterEach` — do not add manual `cleanup()` calls.

## Verification (how to test it)

```text
pnpm -C web test --run
pnpm -C web typecheck
pnpm -C web build
make check
```

Expected: 10 passed (7 scaffold + 3 setup) and exit code 0 for `pnpm -C web test --run`; typecheck 0; build clean; `make check` green. This is the first task whose suite proves a rendered component and a click, not just configuration.

## Out of scope (do not do)

- No `matchMedia`, `IntersectionObserver`, `scrollTo`, `getBoundingClientRect` or `TextEncoder` fakes — add them only if a stage-06/07 test actually fails without them, in that task.
- No global `fetch` stub: each loader test injects its own implementation (05-04), so a global stub would hide real coupling.
- No leaflet import, no App, no component, no dataset fixture, no MSW.

## Definition of done

- [ ] All three cases exist in `web/src/test/setup.test.tsx`, named as written, and pass.
- [ ] Red run for case 2 observed and quoted in the commit body.
- [ ] `pnpm -C web test --run` exits 0 with 10 passing; typecheck, build and `make check` green.
- [ ] `grep -rn 'ResizeObserver' web/src` matches only `setup.ts` and `setup.test.tsx`; no `any` added.
- [ ] Only the two listed files are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled; exactly one commit on the trunk (`main`) with the message below.

## Commit

```text
05-02: prove the jsdom setup with a smoke render

- points: 0.5
- red: setup.test.tsx::test_resize_observer_stub_exists_and_is_a_noop failed (ReferenceError: ResizeObserver is not defined)
- add a global ResizeObserver stub to the shared setup file
- prove jsdom, jest-dom matchers and user-event with a smoke test
- suite: 10 passed
```

Files in the commit: `web/src/test/setup.ts`, `web/src/test/setup.test.tsx`, and this task file.

## Traps and review notes

- A stub that is an object literal passes `typeof === "object"` but fails `new ResizeObserver(...)`; Leaflet constructs it. Assert it is a constructor, as the test does.
- With `globals: false` RTL would not auto-clean and tests would leak DOM between cases; this repo relies on `globals: true` from 05-01 — do not flip it.
- Do not silence a jsdom gap with `vi.mock` inside individual tests; the shared setup is the one place.
- Reviewers: the stub must be installed only when missing (do not clobber a future native implementation); no test file may contain the word `ResizeObserver` outside this task's two files.

## Plan samples are untrusted

The plan says "a ResizeObserver stub" but gives no API or file body. Check the installed jsdom version that the global is genuinely missing, and that `@testing-library/react` performs auto-cleanup when Vitest globals are enabled, before relying on either claim (`CLAUDE.md` §6). Record deviations under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Deviations | |
