---
id: 05-01
stage: 05
title: Vite + React + TS + Tailwind + Vitest scaffold
points: 2
owner: web
status: todo
depends_on: [00-06]
files:
  - web/package.json
  - web/pnpm-lock.yaml
  - web/vite.config.ts
  - web/tsconfig.json
  - web/tsconfig.node.json
  - web/tailwind.config.ts
  - web/postcss.config.js
  - web/index.html
  - web/src/main.tsx
  - web/src/App.tsx
  - web/src/index.css
  - web/src/test/setup.ts
  - web/src/test/scaffold.test.ts
---

# 05-01 · Vite + React + TS + Tailwind + Vitest scaffold

**Points:** 2 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 00-06 · **Unblocks:** 05-02, 05-03, 07-01

## Goal

Stand up the `web/` shell: pinned Vite 5 + React 18 + TypeScript 5, the six Tailwind tokens from `plan/02-architecture.md` §5.3, a Vitest/jsdom entry point, and no template demo noise.

## Why this task exists

Every module in stages 05–08 lives under `web/`; one type-checking build, one jsdom runner and one token set stop later tasks from inventing tooling and drifting from the style contract. It also fixes the command vocabulary (`pnpm -C web test --run`, `pnpm -C web typecheck`, `make check`) that every later task and the reviewer quote.

## Current state (preconditions)

- 00-06 is done: `Makefile` with `test`/`check`; root `.gitignore` covers `node_modules/`, `web/dist/`, `web/test-results/`; `main` is clean.
- No `web/` exists (if `web/package.json` or the lockfile is present, stop and report); Node 22 and pnpm 12.16.1 are installed (`CLAUDE.md` §6).

## Read before you start

- `CLAUDE.md` §1 (one commit), §3 (scaffolding exception), §5 (pinned deps, no `any`), §6, §7; `plan/02-architecture.md` §2 (`web/` tree), §4 (Vite 5, React 18, TS 5, Tailwind 3.4, vitest + RTL + jsdom, pnpm; exact pins, commit the lockfile), §5.3 (tokens, 14 px system UI).
- `plan/stage-05-web-shell.md` Task 5.1; `plan/90-test-matrix.md` rows W-01…W-06 and W-14 — the commands this scaffold must serve.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `web/package.json`, `web/pnpm-lock.yaml` | pinned deps, six scripts, committed lockfile |
| create | `web/vite.config.ts`, `web/tsconfig.json`, `web/tsconfig.node.json` | React plugin + jsdom `test` block; strict TS project |
| create | `web/tailwind.config.ts`, `web/postcss.config.js` | two content globs, six tokens, postcss plugins |
| create | `web/index.html`, `web/src/main.tsx`, `web/src/App.tsx`, `web/src/index.css` | document shell with `root`, mount, empty shell, Tailwind base |
| create | `web/src/test/setup.ts`, `web/src/test/scaffold.test.ts` | jest-dom import only; the seven cases below |

Deleted in the same commit: `web/src/App.css`, `web/src/assets/react.svg`, `web/public/vite.svg`, `web/README.md`, the template's eslint config and packages. Nothing outside `web/` changes; do not edit the root `.gitignore` or the `Makefile`.

## Test first (red)

Scaffolding is the CLAUDE §3 exception: install first, then make the config contract red in `web/src/test/scaffold.test.ts`.

1. `test_tailwind_theme_exposes_the_six_style_tokens` — `theme.extend.colors` is exactly `background #fafafa`, `surface #ffffff`, `border #e5e7eb`, `ink #111827`, `muted #6b7280`, `accent #0f766e`.
2. `test_tailwind_content_globs_cover_index_and_src` — `content` holds `./index.html` and `./src/**/*.{ts,tsx}` and nothing else.
3. `test_dependencies_are_pinned_and_on_the_planned_majors` — no `^`/`~`/`latest`; `react`/`react-dom` 18.x, `vite` 5.x, `typescript` 5.x, `tailwindcss` 3.4.x; `vitest`, `jsdom`, `@testing-library/{react,user-event,jest-dom}`, `postcss`, `autoprefixer`, `@vitejs/plugin-react` pinned too.
4. `test_package_scripts_match_the_task_contract` — `dev`, `build`, `preview` exist; `test` is `vitest run`; `test:watch` is `vitest`; `typecheck` is `tsc --noEmit`; no `e2e` yet.
5. `test_vitest_config_wires_jsdom_globals_and_setup_file` — `environment === "jsdom"`, `globals === true`, `setupFiles` ends in `src/test/setup.ts`.
6. `test_index_html_mounts_the_app_root` — `web/index.html` has an element with id `root` and loads `/src/main.tsx`.
7. `test_demo_assets_are_removed` — `web/src/App.css`, `web/src/assets/react.svg`, `web/public/vite.svg` absent; `App.tsx` mentions neither `vite` nor `counter`.

Expected red symptom: `Cannot find module '../../tailwind.config'`, then `TypeError: Cannot read properties of undefined (reading 'background')`, then `expected 'vitest' to be 'vitest run'`; a syntax error is a broken test, not red. Run: `pnpm -C web test --run` — only this file exists, and the cases stay red until the config lands.

## Implement (green)

- `pnpm create vite web --template react-ts`, then replace the template's ranges with the stack's majors: React 18, ReactDOM 18, Vite 5, TypeScript 5, `@vitejs/plugin-react` 4 (today's template may resolve React 19 / Vite 7 — downgrade; never ship defaults).
- Add `tailwindcss` 3.4, `postcss`, `autoprefixer`, `vitest` (major supporting Vite 5), `jsdom`, `@testing-library/{react,user-event,jest-dom}` and matching `@types`; pin exactly; commit the lockfile. Scripts: `dev` = `vite`, `build` = type-check-then-build, `preview` = `vite preview`, `test` = `vitest run`, `test:watch` = `vitest`, `typecheck` = `tsc --noEmit`.
- `vite.config.ts`: React plugin, no `server.proxy` (static dataset), `test` block with jsdom, `globals: true`, `setupFiles` → `src/test/setup.ts`. Import the helper from `vitest/config`, not `vite`, or `test` fails typecheck.
- `tailwind.config.ts`: the two content globs and the six tokens in `theme.extend.colors`; no plugins. `postcss.config.js`: tailwindcss + autoprefixer only. `index.css`: three directives plus a base layer applying background and ink through the utilities (never a second copy of the hex values).
- `main.tsx` mounts `App` into `root` in StrictMode and imports `index.css`; `App.tsx` stays empty (layout is 05-17/05-18). `test/setup.ts` imports the jest-dom/vitest entry point only.

Run: `pnpm -C web test --run` — seven green; `pnpm -C web typecheck` and `pnpm -C web build` clean. Delete the demo assets listed above and leave no unused template file behind.

## Refactor / keep it DRY

One token table (`tailwind.config.ts`), one test configuration (`vite.config.ts`, no second `vitest.config.ts`), one script set (`package.json`). Delete leftover template files; re-run the suite.

## Verification (how to test it)

```text
pnpm -C web test --run
pnpm -C web typecheck
pnpm -C web build
make check
```

Expected: 7 passed; typecheck 0; `web/dist/` written and gitignored; `make check` green. If the `Makefile` has no web target yet, note it in the commit body and do not edit that file. Optional manual check: `pnpm -C web dev` shows a blank page, clean console, no Vite logo.

## Out of scope (do not do)

- No product components beyond the empty `App.tsx`, no hooks, state, router or `src/types.ts` (05-03+); no ResizeObserver stub (05-02).
- No Playwright/`e2e` (07-13), no `leaflet`/`react-leaflet` (07-01), no `web/public/data/dataset.json`; no eslint/prettier/CI, Tailwind plugins, component library, icon package or extra CSS variables.

## Definition of done

- [ ] All seven cases exist in `web/src/test/scaffold.test.ts`, named as written, and pass; red run quoted in the commit body.
- [ ] `pnpm -C web typecheck` and `pnpm -C web build` clean; lane suite and `make check` green; `grep -rn ': any' web/src` finds nothing; only the listed files are modified (`git status --short` checked, deletions included).
- [ ] Frontmatter `status: done`, `## Result` filled, exactly one commit on the trunk (`main`) with the message below.

## Commit

```text
05-01: scaffold the vite react tailwind web shell

- points: 2
- red: scaffold.test.ts::test_tailwind_theme_exposes_the_six_style_tokens failed (Cannot find module '../../tailwind.config')
- add pinned Vite 5, React 18, TypeScript 5 and Tailwind 3.4 with the six style tokens
- add the vitest/jsdom test block and scripts; delete Vite demo assets
- suite: 7 passed
```

## Traps and review notes

- Template drift: a fresh `pnpm create vite` can pull React 19, Vite 7 and Tailwind 4. Check `pnpm -C web list --depth 0` against §4, downgrade, and put resolved versions in the commit body. Tailwind 4 has no config file/postcss-plugin pair — pin 3.4.x.
- `test` must be `vitest run`: watch mode hangs `make check` forever. Do not add `--passWithNoTests`; the scaffold test proves the harness boots, and a typo in `setupFiles` reddens every later test.
- Reviewers: `grep -rn 'vite.svg\|react.svg\|App.css\|counter' web/src web/index.html` empty; `git ls-files web | grep -c node_modules` 0; lockfile tracked.

## Plan samples are untrusted

The plan describes scripts and tokens but has no usable file bodies. Verify the template's actual file names, the installed Tailwind 3.4 config shape and the vitest option names against `web/node_modules` before relying on them (`CLAUDE.md` §6). Record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Deviations | |
