---
id: 07-13
stage: 07
title: Playwright config and dataset bootstrap
points: 1
owner: web
status: todo
depends_on: [07-09]
files:
  - web/playwright.config.ts
  - web/e2e/global-setup.ts
  - web/src/test/playwright-config.test.ts
  - web/package.json
  - web/pnpm-lock.yaml
  - web/vite.config.ts
  - .gitignore
---

# 07-13 · Playwright config and dataset bootstrap

**Points:** 1 · **Stage:** 07 — Map and selection · **Owner lane:** web · **Depends on:** 07-09 · **Unblocks:** 07-14

## Goal

Stand up the Playwright harness for the map: a preview server on port 4173, a 1440×900 viewport, an `e2e` script, a global setup that materialises the frozen sample dataset where the server serves it, and the ignore rules for Playwright's artifacts.

## Why this task exists

Leaflet drag cannot be tested in jsdom (`plan/02-architecture.md` §4), so the only honest proof of "move the map → the list updates" is a real browser (07-15). Before any spec exists, the harness must be trustworthy: a fixed viewport (the map's 45 vh band must be on screen), no retries (a flaky map test must fail loudly, not pass on attempt three), and a dataset that exists **before** the app loads — the real `web/public/data/dataset.json` is a generated artifact and must never be committed or depended on. This task also keeps Vitest and Playwright from collecting each other's files, which would otherwise break `pnpm -C web test` the moment the first spec lands.

## Current state (preconditions)

- 07-09 done: the app renders the map, the list, the panel and selection from the sample dataset through the loader; `pnpm -C web dev` shows the full stage-07 UI.
- 05-01 done: `web/package.json` scripts (`dev`, `build`, `preview`, `test`, `test:watch`, `typecheck`) and `web/vite.config.ts` with the Vitest block; 05-03 done: `web/src/test/fixtures/dataset.sample.json` (6 units, 3 buildings).
- 00-01 done: the root `.gitignore` already ignores `node_modules/`, `web/dist/`, `web/test-results/` and `web/public/data/`; verify each entry before adding anything.
- Playwright is **not** installed; `web/playwright.config.ts`, `web/e2e/` and `web/src/test/playwright-config.test.ts` do not exist.

## Read before you start

- `CLAUDE.md` §1 (never commit generated artifacts), §3 (a config file may be proven by "the tool runs"), §5 (pinned dependencies), §6.
- `plan/02-architecture.md` §4 — `playwright` is the chosen E2E runner; the "Leaflet in jsdom" paragraph that makes E2E mandatory.
- `plan/stage-07-map.md` Task 7.5 — the config block, the `globalSetup` copy and the `testDir`/`retries` values.
- `plan/90-test-matrix.md` rows W-08, W-13, W-18, W-19 — the commands this harness must serve (`pnpm -C web e2e`).
- `web/package.json`, `web/vite.config.ts`, the root `.gitignore` and `web/src/test/fixtures/dataset.sample.json`.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `web/playwright.config.ts` | webServer, viewport, testDir, retries, globalSetup |
| create | `web/e2e/global-setup.ts` | copy the frozen fixture to `public/data/` and, when present, `dist/data/` |
| create | `web/src/test/playwright-config.test.ts` | the six cases below |
| modify | `web/package.json` | the `e2e` script and the pinned `@playwright/test` devDependency |
| modify | `web/pnpm-lock.yaml` | the lockfile for that pin |
| modify | `web/vite.config.ts` | exclude `e2e/**` from Vitest's test collection |
| modify | `.gitignore` | `web/playwright-report/` (and confirm `web/test-results/`) |

Nothing else may change: no spec file yet (07-14), no `App`/component edit, no `Makefile` edit, no second runner.

## Test first (red)

1. `test_web_server_previews_the_built_app_on_port_4173` — the exported config has `webServer.command` exactly `pnpm preview --port 4173`, `webServer.url` exactly `http://127.0.0.1:4173`, and `reuseExistingServer: false` (a stale server on 4173 must not silently serve an old dataset).
2. `test_viewport_test_dir_and_retries_match_the_contract` — the browser viewport is 1440 wide and 900 high, `testDir` is `e2e`, `retries` is `0`, `globalSetup` points at the file created here.
3. `test_global_setup_materialises_the_sample_dataset` — invoking the setup's default export writes `web/public/data/dataset.json` with content deep-equal to `web/src/test/fixtures/dataset.sample.json` (6 units / 3 buildings); a second invocation is idempotent; the destination is under the gitignored `web/public/data/`, never the fixture.
4. `test_e2e_script_and_pinned_playwright` — `package.json` scripts include `e2e` = `playwright test`; `@playwright/test` is in devDependencies pinned without `^`/`~`; no `cypress`/`puppeteer`/`jest` entry was added.
5. `test_playwright_artifacts_are_gitignored` — the root `.gitignore` contains `web/test-results/` and `web/playwright-report/` (read the file; one entry each).
6. `test_vitest_does_not_collect_the_playwright_specs` — `vite.config.ts`'s `test.exclude` contains `e2e/**`, so `pnpm -C web test --run` will never execute a Playwright spec in jsdom.

Expected red: `Failed to resolve import "../../playwright.config"` for cases 1–2 and `expected undefined to be "playwright test"` for case 4. Run `pnpm -C web test --run playwright-config`, quote one line.

## Implement (green)

- `web/playwright.config.ts`: `testDir` `e2e`, `retries` 0, a 1440×900 browser viewport, `globalSetup` pointing at `./e2e/global-setup.ts`, and the `webServer` block with the three values above plus a startup timeout that is long enough for `vite preview` (a documented constant, not the default guess). No `projects` matrix: one Chromium project is the scope (document it in a comment).
- `web/e2e/global-setup.ts`: a default async export. It reads the frozen fixture and writes it to `web/public/data/dataset.json` after `mkdir -p`, and — because Playwright starts `webServer` **before** `globalSetup` (verified, see the Traps) and `vite preview` serves `web/dist`, not `web/public` — also writes `web/dist/data/dataset.json` when `dist/` exists. When `dist/` is missing, throw one clear error naming the fix (`pnpm -C web build`) instead of letting every test 404 the dataset. Log one line with the destination paths.
- `package.json`: `"e2e": "playwright test"` and `@playwright/test` pinned. The commit body notes that Playwright is already the chosen runner in `plan/02-architecture.md` §4 — no new decision.
- `vite.config.ts`: extend the existing `test` block with the `e2e` exclusion; keep every existing key. Do not create a second Vitest config.
- `.gitignore`: add `web/playwright-report/`; verify `web/test-results/` is already there and add it only if missing.
- Install the browser once locally (`pnpm -C web exec playwright install chromium`) and say so in the commit body or the task `Result`; the browser download is a machine step, not a repository change.

Run: `pnpm -C web test --run playwright-config` — six green. Then `pnpm -C web build` and `pnpm -C web e2e --list`: the config must load without errors. With no specs yet Playwright may report "No tests found" and exit non-zero — record that, and do not add a placeholder spec to make it green (07-14 adds the first real one).

## Refactor / keep it DRY

One config, one setup file, one `e2e` script. The fixture path is a single constant in the setup; the tests import the fixture through `web/src/test/fixtures/…` and never copy it themselves. No second copy of the dataset under `web/e2e/`.

## Verification (how to test it)

```text
pnpm -C web test --run playwright-config
pnpm -C web test --run
pnpm -C web typecheck
pnpm -C web build
pnpm -C web e2e --list
make check
```

Expected: 6 passed for the target; suite and typecheck green; the build succeeds; `--list` loads the config (zero tests is the correct state before 07-14). `make check` must stay offline and must not invoke Playwright — if it does, report it rather than editing the `Makefile` here.

Manual check: `pnpm -C web e2e --list` prints no config error; `web/public/data/dataset.json` exists after running it; `git status --short` shows neither `web/public/data/` nor `web/test-results/`.

## Out of scope (do not do)

- Spec files, drag helpers, tile stubs, popup tests — 07-14…07-16.
- A CI workflow, a Playwright Docker image, trace/video recording, a `projects` matrix, retries above zero.
- Committing `web/public/data/dataset.json` or any other generated artifact.
- A backend, a local API or a second server: `vite preview` serves static files only.
- Replacing Vitest, adding a second unit-test runner, or moving unit tests into Playwright.

## Definition of done

- [ ] All six cases exist, named as written, and pass.
- [ ] Red run observed and quoted; the `globalSetup` case actually executes the setup and compares file content.
- [ ] `pnpm -C web test --run` still collects none of the Playwright files (case 6 proves the exclusion).
- [ ] No generated file is tracked; `.gitignore` covers both Playwright directories.
- [ ] Lane suite, typecheck and build green; `make check` exits 0 and does not run Playwright.
- [ ] `git status --short` lists only the seven files above plus this task file; `status: done`; `## Result` filled; one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
07-13: add the playwright harness and dataset bootstrap

- points: 1
- red: test_e2e_script_and_pinned_playwright failed (expected undefined to be "playwright test")
- add playwright config with preview on 4173, 1440x900 viewport, testDir e2e, retries 0
- copy the frozen sample dataset in globalSetup to public/data and to dist/data when built
- add the e2e script, pin @playwright/test, exclude e2e from vitest, ignore playwright artifacts
- suite: N passed
```

Files in the commit: exactly the seven listed files plus this task file.

## Traps and review notes

- **Ordering:** Playwright launches `webServer` before `globalSetup` (see `microsoft/playwright#31966` and the note in `neomjs/neo#15584`). The plan's "`globalSetup` runs before the server starts" is wrong; the copy must land where `vite preview` serves from (`dist/`), or every test sees a 404 dataset. Record the verified behaviour in `## Result`.
- `vite preview` needs `dist/`: a run on a clean clone without a build fails before any assertion; the setup's error message is the operator's fix.
- `retries: 0` is deliberate: a map test that passes on the second attempt is a flaky test, and 07-15's whole point is measuring the real timing.
- A `reuseExistingServer: true` lets a developer's stray server on 4173 serve a stale build; assert `false`.
- Vitest's default `include` matches `e2e/*.spec.ts`; without the exclusion the unit suite tries to run Playwright specs in jsdom and the first failure looks unrelated.
- `web/public/data/` is gitignored: if `git status` shows it, the copy wrote to the wrong place or the ignore rule was lost.
- Reviewers: grep `web` for `testDir` (one), `port 4173` (one), `dataset.sample.json` in `e2e/` (one), and check `vite.config.ts` still has the jsdom setup file.

## Plan samples are untrusted

The plan's config block is illustrative and its ordering claim is wrong (see Traps). Verify the installed `@playwright/test` option names (`webServer.command`, `webServer.url`, `webServer.reuseExistingServer`, `testDir`, `retries`, `globalSetup`) in `node_modules/@playwright/test` and the docs before writing assertions (`CLAUDE.md` §6). Record every deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
