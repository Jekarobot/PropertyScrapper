---
id: 00-02
stage: 00
title: Repository README skeleton
points: 0.5
owner: pipeline
status: todo
depends_on: [00-01]
files:
  - README.md
---

# 00-02 · Repository README skeleton

**Points:** 0.5 · **Stage:** 00 — Bootstrap · **Owner lane:** pipeline · **Depends on:** 00-01 · **Unblocks:** 08-11

## Goal

Create the repository README with the seven runbook headings from stage 08, as a skeleton the runbook fills in place.

## Why this task exists

There is no front door: a new reader finds `plan/`, `CLAUDE.md` and `tasks/` but no README explaining what the two
programs are. Stage 08 task 8.5 writes the runbook, and doing it in one pass later invites seven uncoordinated edits;
the headings are the contract 08-11 fills and `make help` (00-06) can point at.

## Current state (preconditions)

- 00-01 is done: `.gitignore` exists, `data/.gitkeep` is tracked and `git status --short` shows nothing generated.
- `README.md`, `pipeline/`, `web/` and `Makefile` do not exist yet; no virtualenv is needed. The only source of the
  section list is `plan/stage-08-hardening-ops.md` Task 8.5.

## Read before you start

- `plan/stage-08-hardening-ops.md` Task 8.5 — the seven sections, each 3–10 lines once filled.
- `plan/README.md` §0–§1 — what the product is, as facts to summarise in two sentences, not prose to copy.
- `CLAUDE.md` §7 (planning documents contain no code) and §11 (the authoritative command list to point at).

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `README.md` | title, two-sentence description, seven placeholder sections |

Nothing else may change; if another file seems required, stop and report.

## Test first (red)

No pytest file: pytest arrives with 00-03 and this is a documentation scaffold (`CLAUDE.md` §3 exception). Red is
observed with named shell checks run from the repo root.

1. `check-1 test -f README.md` — today exits `1`; after, `0`; checks 2–5 have nothing to read until it exists.
2. `check-2 head -1 README.md` — after, prints `# Connex Watch`.
3. `check-3 grep -c '^## ' README.md` — after, prints `7`.
4. `check-4 grep -n '^## ' README.md` — after, prints exactly `## What it is`, `## Quick start`, `## Data freshness`, `## Exit codes`, `## Troubleshooting`, `## Roadmap`, `## Ethics`, in that order.
5. `check-5 grep -c 'make check\|pnpm -C web\|connex-watch crawl' README.md` — after, prints `0`: no command that cannot run yet, and every section holds one non-empty placeholder sentence.

Expected red: `test -f README.md; echo $?` prints `1`; `grep` and `head` fail on a missing file.

## Implement (green)

Line 1 is `# Connex Watch`. Then one paragraph, two sentences: a scheduled Python pipeline crawls the public
connex.in.th rent search pages, keeps only cards whose state is `available`, and publishes one static `dataset.json`;
the viewer is a Vite + React + Leaflet SPA with filters, map and list, no backend and no database. Then one status line
naming `CLAUDE.md` §11 as the authoritative command list until the runbook lands.

Add exactly seven `##` sections in the order above, each with one placeholder sentence naming what 08-11 will document:
no bullet lists, tables or fenced blocks inside them, no badges, screenshot, licence, marketing or emoji.

Constraints: the fixtures under `pipeline/tests/fixtures/` stay untouched and frozen, no command in this task reaches
the network, and only `README.md` is committed.

## Refactor / keep it DRY

The heading strings and the two-sentence description are the single source for 08-11 and `make help`: do not repeat
the seven section names elsewhere, and do not add a second "how to run" paragraph.

## Verification (how to test it)

```text
test -f README.md
grep -n '^## ' README.md
grep -c 'make check\|pnpm -C web\|connex-watch crawl' README.md
git status --short
```

Expected: `grep -n` prints the seven headings in order; the forbidden count is `0`; `git status --short` lists only `README.md`.

## Out of scope (do not do)

- The runbook body, exit-code values, troubleshooting entries, roadmap prose and `docs/screenshot.png` — 08-11.
- The `Makefile` and its `help` text (00-06), `pipeline/**` and `.venv` (00-03), anything under `web/` (05-01).
- `LICENSE`, `CONTRIBUTING.md`, `CHANGELOG.md`, `CODE_OF_CONDUCT.md`, a `docs/` directory, CI badges, translations;
  describing commands as working before they exist: `make check`, `pytest pipeline`, `pnpm -C web`, `connex-watch`.

## Definition of done

- [ ] Checks 1–5 behave as written; the heading order and the forbidden-command count are exact.
- [ ] Red observed and quoted (`test -f README.md` exited `1`); only `README.md` is created.
- [ ] `status: done`; `## Result` filled; exactly one commit on the trunk with the message below.

## Commit

```text
00-02: add the repository readme skeleton

- points: 0.5
- red: test -f README.md exited 1 (no such file)
- add the title, a two-sentence description and the seven runbook headings, each a one-line placeholder
- no command that cannot run yet
- suite: n/a — pytest arrives in 00-03; checks 1-5 green
```

Files in the commit: exactly `README.md` and this task file (status and `## Result`). Committed alone on the trunk
(`main`): one task = one commit, no push, no branch.

## Traps and review notes

- Do not write a quick start that fails today: `make check` (00-06), `pytest pipeline` (00-03), `pnpm -C web` (05-01)
  and `connex-watch` (02-13) do not work yet — point at `CLAUDE.md` §11. Do not copy plan prose as a block.
- Exactly seven `##` headings and no empty one; no "Architecture", "API" or "Development" section.
- Reviewer: `grep -c '^## ' README.md` is `7`, the order matches, no fenced block, and `git status` shows only README.

## Plan samples are untrusted

Task 8.5 lists section names, not heading text or prose, and any `plan/` snippet is illustrative: apply `CLAUDE.md`
§6, verify the facts against `plan/README.md` §0–§1 and the commands against §11 of `CLAUDE.md`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
