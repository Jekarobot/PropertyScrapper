---
id: 05-06
stage: 05
title: Abort signal support in the loader
points: 0.5
owner: web
status: todo
depends_on: [05-04]
files:
  - web/src/api/loadDataset.ts
  - web/src/api/loadDataset.test.ts
---

# 05-06 · Abort signal support in the loader

**Points:** 0.5 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 05-04 · **Unblocks:** 05-13

## Goal

Give `loadDataset` an optional third `abortSignal`: a pre-aborted signal rejects immediately with an `AbortError`-shaped error and the signal is forwarded to the fetch implementation.

## Why this task exists

`useDataset` (05-13) loads in an effect and must not let a response from an unmounted component overwrite state, nor make a second request while the first is still in flight. React gives the hook a cleanup function, so the loader must accept a signal and — importantly — reject with a distinguishable error rather than a `DatasetError`: an abort is not a data failure and must not render the error screen. Today the loader takes two arguments and would call `fetch` even when the caller already gave up.

## Current state (preconditions)

- 05-04 is done: `loadDataset(url = "/data/dataset.json", fetchImpl = fetch)` resolves for `schemaVersion` 1 and throws `DatasetError` otherwise; ten loader cases exist after 05-05.
- The loader calls the fetch implementation with the url only, never with an init object, and has no abort handling.
- The test file imports the fixture and injects every fetch implementation.

## Read before you start

- `CLAUDE.md` §3 (fake fetcher, observed red), §5 (YAGNI), §6, §7.
- `plan/stage-05-web-shell.md` — Task 5.2's signature and the `abortSignal` line in the TDD list.
- `plan/02-architecture.md` §3.4 — one file, one fetch; §5.2 F10 is unrelated but confirms no URL sync.
- `plan/90-test-matrix.md` row W-01 — the loader contract this extends.
- `web/src/api/loadDataset.ts` — read where the fetch call happens before adding a parameter.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/api/loadDataset.ts` | third optional parameter, pre-check, forwarded signal |
| modify | `web/src/api/loadDataset.test.ts` | the three cases below |

Nothing else may change: no hook, no component, no dependency, no `types.ts` or fixture edit.

## Test first (red)

Append three cases to `web/src/api/loadDataset.test.ts`. Use a real `AbortController`; do not hand-build a signal-like object.

1. `test_pre_aborted_signal_rejects_without_calling_fetch` — create a controller, call `controller.abort()`, then call `loadDataset(undefined, fake, controller.signal)`; the promise rejects with an error whose `name === "AbortError"` and which is **not** a `DatasetError`; the fake records zero calls.
2. `test_signal_is_forwarded_to_the_fetch_implementation` — a live controller; after the call resolves, the fake's second argument is an init object whose `signal` is the same controller signal. Also assert the url is still the first argument.
3. `test_fetch_rejection_with_abort_error_propagates_unchanged` — the fake rejects with a specific abort-shaped error; the loader rejects with that same object (same `name`, same message, not wrapped in `DatasetError`), so the hook can tell an abort from a failure by identity and name.

Expected red symptom: case 1 fails with `expected "fetch" to not be called at all, but actually been called 1 times` (the loader ignores the extra argument and fetches); case 2 fails with `expected undefined to equal [object AbortSignal]`. If a case already passes, it does not exercise the change — fix the case.

Run: `pnpm -C web test loadDataset` — three new cases red, ten old cases green.

## Implement (green)

- Extend the public signature to `loadDataset(url = "/data/dataset.json", fetchImpl = fetch, abortSignal?: AbortSignal)`; the docstring gains one clause: an aborted signal rejects with an `AbortError`-named error and never resolves with data.
- If `abortSignal?.aborted` is true before the request, return a rejected promise with an `AbortError`: construct a `DOMException` with name `AbortError` (Node 22 and jsdom both provide it), or forward the signal's own `reason` when it already is one. Do not call the fetch implementation in that path.
- Forward the signal to the fetch implementation as its second argument **only when a signal was supplied** (`{ signal: abortSignal }`); with no signal, keep calling with the url alone so the 05-04 call-argument case stays green.
- Do not catch a fetch rejection: an `AbortError` from `fetch` must propagate unchanged, and everything else keeps the 05-05 behaviour. Do not create an internal `AbortController`, timeout or retry.
- No `any`: type the third parameter as `AbortSignal | undefined`; access `signal.reason` only after narrowing it.

Run: `pnpm -C web test loadDataset` — thirteen green.

## Refactor / keep it DRY

One abort check, at the top of `loadDataset`; the hook must not re-implement an "already aborted" guard. One error-classification rule: `AbortError` stays outside `DatasetError` and every consumer tests `name`, not `instanceof DOMException` (jsdom and Node may expose different constructors). Test 3 reuses the same error instance through the fake instead of reconstructing an abort.

## Verification (how to test it)

```text
pnpm -C web test loadDataset
pnpm -C web test --run
pnpm -C web typecheck
pnpm -C web build
make check
```

Expected: 13 passed for the target, 30 for the lane, typecheck 0, build clean, `make check` green. Manual: `grep -n 'AbortError' web/src/api/loadDataset.ts` shows the pre-check and no `catch` branch that converts it.

## Out of scope (do not do)

- No timeout, retry, request cancellation on a timer, `AbortController` inside the loader, or dedupe of concurrent requests.
- No React hook, effect, cleanup wiring (05-13); no UI change; no `DatasetError` subclass for aborts.
- No test of a mid-flight abort driven by the real `fetch` — the injected fake proves forwarding, and the real network is never used.

## Definition of done

- [ ] All three cases exist, named as written, and pass; the ten earlier loader cases still pass untouched.
- [ ] Red run observed and quoted; the global `fetch` was never called in any case.
- [ ] `pnpm -C web typecheck` clean; lane suite and `make check` green; no `any`.
- [ ] Only the two listed files are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled; exactly one commit on the trunk (`main`) with the message below.

## Commit

```text
05-06: accept an abort signal in the dataset loader

- points: 0.5
- red: loadDataset.test.ts::test_pre_aborted_signal_rejects_without_calling_fetch failed (expected "fetch" to not be called)
- reject a pre-aborted signal with an AbortError before fetching
- forward the signal to the injected fetch implementation
- suite: 30 passed
```

Files in the commit: `web/src/api/loadDataset.ts`, `web/src/api/loadDataset.test.ts`, and this task file.

## Traps and review notes

- `{ signal: undefined }` as a second argument breaks the earlier "called with the url" assertion; build the init object conditionally.
- Do not wrap the abort in `DatasetError`: 05-13 must keep the previous data on screen when its effect cleanup fires, and an error screen on unmount is the bug this task prevents.
- Do not rely on `error instanceof DOMException`: the abort-shaped error may come from the caller, Node or jsdom. Test `name`.
- Reviewers: `grep -n 'abortSignal' web/src/api/loadDataset.ts` shows one parameter use in the pre-check and one in the init; no `setTimeout`, no internal controller.

## Plan samples are untrusted

The plan's signature lists only two parameters; the abort parameter is specified in prose in Task 5.2 and in this task. Verify `DOMException` and `AbortController` availability under the installed jsdom and Node 22 (`CLAUDE.md` §6), and do not copy a body from the plan. If the plan sample's signature is taken literally, this task's third parameter wins; record the deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Deviations | |
