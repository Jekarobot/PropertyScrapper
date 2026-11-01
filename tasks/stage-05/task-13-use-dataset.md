---
id: 05-13
stage: 05
title: Dataset loading hook with loading, ready and error states
points: 1
owner: web
status: todo
depends_on: [05-06]
files:
  - web/src/state/useDataset.ts
  - web/src/state/useDataset.test.ts
---
# 05-13 · Dataset loading hook with loading, ready and error states
**Points:** 1 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 05-06 · **Unblocks:** 05-17
## Goal
Add `useDataset`: one hook that loads `dataset.json` through the typed loader and exposes one of three states
plus `reload()`.
## Why this task exists
The app renders from one published `dataset.json` (`plan/02-architecture.md` §3.3). Without this hook each panel
would fetch on its own and nothing would know whether data is loading, loaded or failed — the three states stage
06 must render. Every later task reads it; no component may call `loadDataset` directly.
## Current state (preconditions)
- 05-01/05-02 done: Vite + React + TS + Vitest + jsdom wired, `web/src/test/setup.ts` present, web tests green.
- 05-03 done: `web/src/types.ts` has `Dataset`; the sample fixture (6 units / 3 buildings) and
  `web/src/test/fixtures.ts` exist.
- 05-04/05-05 done: `web/src/api/loadDataset.ts` exports `loadDataset(url?, fetchImpl?)` → `Promise<Dataset>` and
  `DatasetError`; 05-06 done: it accepts an abort signal — read that file, do not edit it. No `web/src/state/`
  yet.
## Read before you start
- `CLAUDE.md` §3 (TDD; unit tests never touch the network — `vi.stubGlobal("fetch", …)`), §5 (no `any` in
  `web/src/state/**`).
- `plan/stage-05-web-shell.md` Task 5.4 — hook signature and state names; `plan/02-architecture.md` §3.3 — the
  fields asserted below.
- `plan/90-test-matrix.md` rows W-05/W-14; `web/src/api/loadDataset.ts` and its test — contract and stub style.
## Files to touch
| Action | Path | What goes there |
|---|---|---|
| create | `web/src/state/useDataset.ts` | the hook and its return type |
| create | `web/src/state/useDataset.test.ts` | the six cases below |

Only these two files change; report anything else you believe is needed.
## Test first (red)
Write these tests before creating `useDataset.ts`; every test stubs `fetch`, none touches the network.

1. `useDataset.test.ts::test_initial_state_is_loading_and_fetches_once` — stub returns the sample dataset →
   `status` `"loading"`, data and error undefined, one call to `/data/dataset.json`.
2. `useDataset.test.ts::test_resolves_to_ready_with_the_sample_dataset` — after settling: `"ready"`,
   `data.schemaVersion === 1`, `data.stats.units === 6`, `data.buildings.length === 3`.
3. `useDataset.test.ts::test_error_path_exposes_the_loader_message` — HTTP 500 → `"error"`, data undefined, error
   is the message string containing `500`, not an `Error` object.
4. `useDataset.test.ts::test_reload_transitions_back_to_loading_then_ready` — from `ready`, `reload()` sets
   `"loading"` and starts a second fetch; on settle the data is the second payload.
5. `useDataset.test.ts::test_reload_after_error_recovers` — first attempt fails, second succeeds → error undefined
   again; a failure is not sticky.
6. `useDataset.test.ts::test_unmount_aborts_the_in_flight_request` — keep the fetch pending, capture its signal,
   unmount → `signal.aborted === true`; the late resolution neither warns nor throws.

Expected red: `Failed to resolve import "./useDataset" from "src/state/useDataset.test.ts"` for every case; run
`pnpm -C web test --run useDataset` and copy one failure line.
## Implement (green)
Create `web/src/state/useDataset.ts` with one hook and one exported type:

- `useDataset(url?: string)` → `{ status, data, error, reload }`; `status` is `"loading" | "ready" | "error"`,
  `data` is `Dataset | undefined`, `error` is `string | undefined`, `reload(): void`; default URL
  `/data/dataset.json`.
- On mount and each `reload()`: fresh `AbortController`, set loading, call the loader with that signal; success →
  `ready` plus data; rejection → silent when aborted or unmounted, else `error` with the thrown error's message.
- Unmount aborts and marks the result stale, so no state is set later; `reload` keeps a stable identity;
  one-sentence docstring. No retries, polling or caching.

Run: `pnpm -C web test --run useDataset` — six green.
## Refactor / keep it DRY
The fetch-once, abort-and-ignore-stale logic exists once: never copy it into `App.tsx` or a second hook. Export
the status union so `App.tsx` switches on it instead of re-declaring strings.
## Verification (how to test it)
```text
pnpm -C web test --run useDataset
pnpm -C web test --run
pnpm -C web typecheck
make check
```

Expected: 6 passed, suite green, `tsc --noEmit` clean, `make check` green; manual `pnpm -C web dev` shows one
request.
## Out of scope (do not do)
- Rendering: no JSX in the hook, no component, no `App.tsx` change — that is 05-17.
- Editing `web/src/api/loadDataset.ts` or its tests: 05-06 fixed that contract.
- Retry/backoff, caching, `localStorage`, polling, a state library, an options bag, URL sync (F10).
## Definition of done
- [ ] All six cases exist, named as written, and pass.
- [ ] Red run observed and quoted in the commit body; target tests + lane suite + typecheck green.
- [ ] No `any` in the hook; only the two listed files plus this task file are modified.
- [ ] `status: done`, `## Result` filled; exactly one trunk commit with the message below.
## Commit
Message (first line exactly, then body):

```text
05-13: add dataset loading hook

- points: 1
- red: test_initial_state_is_loading_and_fetches_once failed (Failed to resolve import "./useDataset")
- add useDataset with loading/ready/error states, a fresh AbortController, reload()
- abort on unmount; ignore aborted and late results
- suite: <N> passed
```

Commit only the listed files plus this task file.
## Traps and review notes
- Returning the `Error` object instead of its message breaks the `error?: string` contract; the UI prints
  `[object Error]` or nothing.
- One `AbortController` reused across `reload()` calls aborts its own second fetch; swallowing abort errors into
  the error state turns every unmount into a phantom banner.
- Reviewers: grep for `fetch(` outside `loadDataset.ts` (none) and for `AbortController` outside this hook (none
  yet).
## Plan samples are untrusted
The signature in `plan/stage-05-web-shell.md` Task 5.4 is illustrative. Apply `CLAUDE.md` §6: check the React
version in `web/package.json`, the real `loadDataset` signature and the installed `@testing-library/react` types
for `renderHook`/`waitFor`. If a sample disagrees with the repo, follow the repo and record it under
`## Decisions`. Add no dependency.
## Decisions

## Result
| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
