---
id: 05-04
stage: 05
title: Typed dataset loader, happy path
points: 1
owner: web
status: todo
depends_on: [05-03]
files:
  - web/src/api/loadDataset.ts
  - web/src/api/loadDataset.test.ts
---

# 05-04 · Typed dataset loader, happy path

**Points:** 1 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 05-03 · **Unblocks:** 05-05, 05-06, 05-13

## Goal

Add `loadDataset` — one function that fetches the published dataset, parses it, checks the schema version and returns a typed `Dataset`, with a `DatasetError` type for every failure it owns.

## Why this task exists

The app has exactly one data source: a static `dataset.json` written by the pipeline. Every later layer — hooks, filters, map, status bar — receives the parsed `Dataset` from this function, so its signature and its error type are the contract the rest of stage 05 is written against. Doing it in one place means the UI never sees raw JSON, and a version mismatch fails loudly instead of rendering `undefined` prices. This task deliberately implements only the happy path plus the version guard; HTTP, JSON and shape failures get their own tests in 05-05, and abort in 05-06.

## Current state (preconditions)

- 05-03 is done: `web/src/types.ts` exports `Dataset` (and the nested interfaces), the sample fixture lives at `web/src/test/fixtures/dataset.sample.json`, and `types.test.ts` is green.
- 05-02 is done (it is on the lane path before this task): the jsdom setup, jest-dom and user-event work.
- `web/src/api/` does not exist. No `loadDataset`, no hook, no component.

## Read before you start

- `CLAUDE.md` §3 (fake fetcher, no network in unit tests), §5 (no `any`, YAGNI), §6, §7.
- `plan/stage-05-web-shell.md` Task 5.2 — the signature and the six-loader-case list (cases 3–6 are 05-05/05-06).
- `plan/02-architecture.md` §3.3 — `schemaVersion` and the fields the loader returns; §3.4 — one file, one fetch.
- `plan/90-test-matrix.md` row W-01 — "dataset loader validates schema/version/shape".
- `web/src/types.ts` and the committed fixture — read the actual field names; never retype them from the plan.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `web/src/api/loadDataset.ts` | `loadDataset` + `DatasetError`; no other exports |
| create | `web/src/api/loadDataset.test.ts` | the five cases below |

Nothing else may change: no hook, no component, no `types.ts` edit, no fixture edit.

## Test first (red)

Write `web/src/api/loadDataset.test.ts` before the module. Import the committed fixture and build payload variants by spreading it; never retype the dataset. Inject a fake `fetchImpl` in every case — no `vi.stubGlobal` for the happy path, and no case may reach the network.

1. `test_loadDataset_resolves_with_the_sample_dataset` — the fake returns a 200 response whose `json()` resolves to the fixture; the result has `schemaVersion === 1`, three buildings, `buildings[0].id === "p00348"` and a `stats` block deep-equal to the fixture's.
2. `test_loadDataset_uses_the_default_url` — called with `undefined` url and the fake as the second argument; the fake is called exactly once with `/data/dataset.json`. Passing `undefined` keeps the default while proving no global fetch is touched.
3. `test_loadDataset_uses_the_injected_fetch_instead_of_the_global` — `vi.stubGlobal("fetch", …)` with a spy that throws; the loader with the injected fake resolves and the global spy records zero calls. This is the guarantee every later test relies on.
4. `test_unsupported_schema_version_rejects_with_schemaVersion_in_message` — payload `{...fixture, schemaVersion: 2}` → rejects with a `DatasetError` whose `message` contains `schemaVersion`, whose `name` is `DatasetError`, and which is not a `SyntaxError`.
5. `test_datasetError_is_an_error_subclass_with_detail` — `new DatasetError("boom", "/data/dataset.json")` is an `Error`, has `name === "DatasetError"`, `message === "boom"`, `detail === "/data/dataset.json"`; `new DatasetError("boom")` has `detail === undefined`.

Expected red symptom: `Cannot find module './loadDataset'` / `Failed to resolve import './loadDataset'`. If a case fails with a syntax error, fix the test first.

Run: `pnpm -C web test loadDataset` — all five red for the missing module.

## Implement (green)

- `DatasetError extends Error`: constructor takes a message and an optional `detail: string`; it sets `name` to `DatasetError` itself (subclassing alone leaves `Error`). `detail` is a short machine-friendly hint — URL, status or field path — never a stack or a raw payload.
- `loadDataset(url = "/data/dataset.json", fetchImpl = fetch): Promise<Dataset>`: call `fetchImpl(url)` once, await `response.json()`, narrow the parsed value to `unknown`, and return it as a `Dataset` only when `schemaVersion === 1`. Any other version throws a `DatasetError` whose message contains `schemaVersion` and the received value, with `detail` set to the url.
- Do **not** check `response.ok`, guard `json()` or validate the other fields in this task — 05-05 adds those tests first. Do not cache, retry, transform, sort or mutate the payload; return exactly what the file says.
- Keep the default parameter in the signature (`fetchImpl = fetch`) so a test or the hook can inject an implementation. Do not bind the global fetch at module scope, and do not add a third parameter yet (05-06 owns `abortSignal`).
- One-sentence docstring on `loadDataset` (input → parsed dataset → error) and on `DatasetError`. No `any` in this file: `response.json()` is typed `any` by the DOM lib, so assign it to `unknown` immediately.

Run: `pnpm -C web test loadDataset` — five green.

## Refactor / keep it DRY

One supported-version check, inside `loadDataset`; no second copy in a hook or component. One error class for the whole data layer; 05-05 and 05-13 must throw and catch `DatasetError`, not invent variants. Tests build variants by spreading the imported fixture rather than duplicating its six units.

## Verification (how to test it)

```text
pnpm -C web test loadDataset
pnpm -C web test --run
pnpm -C web typecheck
pnpm -C web build
make check
```

Expected: 5 passed for the target, 22 for the lane, typecheck 0, build clean, `make check` green. Manual: `grep -rn 'fetch(' web/src` shows the call only inside `loadDataset.ts`.

## Out of scope (do not do)

- No HTTP-status, malformed-JSON or shape checks (05-05); no `abortSignal` (05-06); no retry, timeout, cache or dedupe.
- No React hook, component, App wiring (05-13/05-17); no runtime schema library; no state management.
- No edit to `types.ts` or the fixture; no new dependency.

## Definition of done

- [ ] All five cases exist in `web/src/api/loadDataset.test.ts`, named as written, and pass.
- [ ] Red run observed and quoted; no case touches the network or the global fetch (except the stubbed spy in case 3).
- [ ] `pnpm -C web typecheck` clean; lane suite and `make check` green; no `any` in the module.
- [ ] Only the two listed files are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled; exactly one commit on the trunk (`main`) with the message below.

## Commit

```text
05-04: add the typed dataset loader happy path

- points: 1
- red: loadDataset.test.ts::test_loadDataset_resolves_with_the_sample_dataset failed (Cannot find module './loadDataset')
- add loadDataset with an injectable fetch and a schemaVersion guard
- add DatasetError with a detail field
- suite: 22 passed
```

Files in the commit: `web/src/api/loadDataset.ts`, `web/src/api/loadDataset.test.ts`, and this task file.

## Traps and review notes

- Do not capture `fetch` at module load (`const http = fetch`): the test stubs the global and the injection would silently keep the original. Keep the default parameter.
- Do not return the parsed JSON before the version check, and do not cast with `as Dataset` as the only validation.
- `response.json()` is `any`; assign to `unknown` so `tsc --noEmit` and the no-`any` rule both hold.
- Reviewers: `grep -n 'schemaVersion' web/src/api/loadDataset.ts` must show one comparison; `grep -n 'dataset.sample' web/src/api/loadDataset.test.ts` proves the fixture is the payload source.

## Plan samples are untrusted

The plan's Task 5.2 shows the signature and the error class in a code block. Verify against the installed TypeScript/undici typings that `fetch` and `Response` are available under the jsdom environment, and do not copy the sample's body (`CLAUDE.md` §6). If the sample's parameter list conflicts with this task, the task wins; record deviations under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Deviations | |
