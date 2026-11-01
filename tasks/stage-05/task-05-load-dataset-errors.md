---
id: 05-05
stage: 05
title: "Loader errors: schema, HTTP, JSON, shape"
points: 1
owner: web
status: todo
depends_on: [05-04]
files:
  - web/src/api/loadDataset.ts
  - web/src/api/loadDataset.test.ts
---

# 05-05 · Loader errors: schema, HTTP, JSON, shape

**Points:** 1 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 05-04 · **Unblocks:** 05-13

## Goal

Make `loadDataset` fail as one predictable `DatasetError` for a non-OK response, malformed JSON and a structurally invalid payload — never a bare `SyntaxError`, `TypeError` or a resolved half-dataset.

## Why this task exists

A failed nightly run, a half-published file or a proxy error page must produce the error screen, not a crash in the middle of rendering. The web app can only show "could not load data, retry" because the loader reports a clean failure; without this task, `fetch`'s 500 body would reach `JSON.parse`, an HTML error page would throw `SyntaxError`, and a missing `buildings` array would surface later as `undefined.map`. `useDataset` (05-13) maps `DatasetError` to the error state and the abort error to silence, so the distinction matters beyond this file.

## Current state (preconditions)

- 05-04 is done: `web/src/api/loadDataset.ts` exports `loadDataset(url, fetchImpl)` and `DatasetError`; the version guard rejects `schemaVersion !== 1`.
- `web/src/api/loadDataset.test.ts` has five green cases; the fixture is imported there and no test touches the network.
- The loader still returns for non-OK responses, lets `json()` throw raw, and does not look at `stats` or `buildings`.

## Read before you start

- `CLAUDE.md` §3 (observe the red, no weakened assertions), §5 (no `any`, YAGNI), §6, §7.
- `plan/stage-05-web-shell.md` Task 5.2 cases 3–5 — the exact failure list this task implements.
- `plan/02-architecture.md` §3.3 — `stats` and `buildings` are required; §3.4 — one file, fetched once.
- `plan/90-test-matrix.md` row W-01; row W-14 — the error state later renders these failures.
- `web/src/api/loadDataset.ts` — read the current order of operations before changing it.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `web/src/api/loadDataset.ts` | status check, JSON guard, `validateDataset` shape checks |
| modify | `web/src/api/loadDataset.test.ts` | the five new cases below |

Nothing else may change: no hook, no component, no `types.ts` or fixture edit, no new dependency.

## Test first (red)

Append to `web/src/api/loadDataset.test.ts`. Build payloads by spreading the imported fixture; every failure case asserts the error class and the message substring, not the whole string.

1. `test_non_ok_response_rejects_with_status` — fake response `{ ok: false, status: 500 }` → rejects with `DatasetError` whose message contains `500`, whose `detail` is a non-empty string, and which is not a `SyntaxError`.
2. `test_malformed_json_rejects_with_DatasetError_not_SyntaxError` — `json()` rejects with `SyntaxError("Unexpected token <")` (the proxy's HTML page) → rejects with `DatasetError`; message contains `JSON`; `detail` carries the parser text; `expect(error).not.toBeInstanceOf(SyntaxError)`.
3. `test_missing_buildings_rejects_naming_the_field` — payload with `buildings` deleted → `DatasetError`; message contains `buildings`; detail is the url. Same expectation when `buildings` is present but not an array, and when the whole payload is `null` (the message then names the dataset root instead of throwing a `TypeError`).
4. `test_missing_stats_rejects_naming_the_field` — payload with `stats` deleted → `DatasetError`; message contains `stats`; detail is the url.
5. `test_non_finite_stats_counts_reject` — table-driven over every required counter: `stats.units` as `NaN`, as `Infinity` and as the string `"6"`, plus one non-finite/wrong-typed value each for `stats.buildings`, `stats.buildingsWithCoords`, `stats.pagesScanned` and `stats.duplicatesDropped` → each rejects with a `DatasetError` naming that exact path (e.g. `stats.units`). `JSON.parse("1e999")` really does produce `Infinity`, so this is not theoretical.

Expected red symptom: `expected promise to reject with DatasetError` / `expected [Function] to throw DatasetError but got SyntaxError` / `expected 'invalid dataset' to contain 'stats.units'`. If a new case already passes, it is not red — check that it exercises the guard you are about to add.

Run: `pnpm -C web test loadDataset` — five new cases red, the five old ones still green.

## Implement (green)

Keep the public contract unchanged (`loadDataset(url, fetchImpl): Promise<Dataset>`, `DatasetError`) and change only the pipeline inside:

- Check `response.ok` before reading the body; on failure throw `DatasetError` naming `HTTP` and the status, with `detail` = url. This runs first so an HTML error page is never reported as a JSON problem.
- Read the body inside a `try`/`catch`: a rejected `json()` becomes a `DatasetError` whose message contains `JSON` and whose `detail` is the parser's message. Never rethrow `SyntaxError`.
- Introduce one module-private `validateDataset(payload, url): Dataset` that replaces the inline version check: payload must be a non-null object (else message names the root); `buildings` must be an array (message contains `buildings`); `stats` must be a non-null object (message contains `stats`); `stats.units`, `stats.buildings`, `stats.buildingsWithCoords`, `stats.pagesScanned` and `stats.duplicatesDropped` must be finite numbers (message names the exact path, e.g. `stats.units`); `schemaVersion` must be `1` (message contains `schemaVersion`); only then return the narrowed value as `Dataset`.
- One error prefix for shape problems (e.g. `invalid dataset:`) so the UI can show the message as-is. `detail` stays the url for shape failures; do not paste the payload into a message.
- Keep the docstring accurate: it now lists the three failure classes in one sentence. No `any`; `response.json()` goes straight into `unknown`.

Run: `pnpm -C web test loadDataset` — ten green.

## Refactor / keep it DRY

All validation lives in `validateDataset`, called once; the version check moves into it instead of staying duplicated in the fetch body. No second error type: `DatasetError` covers HTTP, JSON, shape and version; only abort (05-06) is a different error on purpose. Tests share one payload-variant helper built on the imported fixture, not four hand-typed datasets.

## Verification (how to test it)

```text
pnpm -C web test loadDataset
pnpm -C web test --run
pnpm -C web typecheck
pnpm -C web build
make check
```

Expected: 10 passed for the target, 27 for the lane, typecheck 0, build clean, `make check` green. Manual: `grep -rn 'SyntaxError' web/src/api` prints nothing outside the test's rejected stub.

## Out of scope (do not do)

- No abort handling (05-06), no retry, timeout, cache, or fallback URL.
- No validation of every §3.3 field: only the version, `buildings`, `stats` and the numeric stats counters are proven here. `units[]` contents are trusted because the pipeline validates them (stage 03).
- No toast, error component or hook (05-13/05-17); no logging; no `console.error`.

## Definition of done

- [ ] All five new cases exist, named as written, and pass; the five old cases still pass unchanged.
- [ ] Red run observed and quoted; no assertion weakened to fit a passing implementation.
- [ ] `pnpm -C web typecheck` clean; lane suite and `make check` green; no `any`.
- [ ] Only the two listed files are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled; exactly one commit on the trunk (`main`) with the message below.

## Commit

```text
05-05: fail the dataset loader as one typed error

- points: 1
- red: loadDataset.test.ts::test_malformed_json_rejects_with_DatasetError_not_SyntaxError failed (expected promise to reject with DatasetError)
- check response.ok before parsing and name the HTTP status
- wrap json() failures and validate buildings/stats in one validateDataset
- suite: 27 passed
```

Files in the commit: `web/src/api/loadDataset.ts`, `web/src/api/loadDataset.test.ts`, and this task file.

## Traps and review notes

- Order matters: status first, then JSON, then shape, then version. Parsing the 500 body first turns an outage into a JSON error and sends the user to the wrong screen.
- Do not "fix" a missing `stats` by defaulting it to zeros: a partial dataset must fail loudly, not render as a city with no units.
- Do not catch the `DatasetError` you throw inside `validateDataset` and re-wrap it; the message would nest twice.
- `JSON.parse("1e999")` yields `Infinity`, so a finite-number check is real, not defensive noise.
- Reviewers: `grep -n 'isFinite\|Number.isFinite' web/src/api/loadDataset.ts` exists; `grep -n 'catch' web/src/api/loadDataset.ts` wraps only `json()`.

## Plan samples are untrusted

The plan lists the failure cases but no implementation. Verify against the installed TypeScript and undici typings that `Response.json()` is `Promise<any>` (hence the `unknown` narrowing) and that a rejected `json()` surfaces as `SyntaxError`; do not copy any body from a plan snippet (`CLAUDE.md` §6). Record deviations under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Deviations | |
