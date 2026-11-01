# 91 — Sub-agent briefs, ownership and sequencing

How to split the implementation across agents so that two of them never touch the same file, and so
that a reviewer can prove each stage is done in ten minutes.

---

## 1. Ownership matrix

| Path | Owner | Stages |
|---|---|---|
| `plan/**` | architect (read-only for agents) | — |
| `Makefile`, `.gitignore`, root `README.md` | agent A | 00, 08 |
| `pipeline/**` | **agent A only** | 00–04 |
| `data/**` (not committed) | agent A | 02–04 |
| `web/**` | **agent B only** | 05–07 |
| `scripts/**` | whoever runs 08 | 08 |
| `web/e2e/**` | agent B | 07, 08 |

**Rule:** one writer per path. If a stage needs a file owned by another agent, the other agent makes
that change in its own commit and the dependency is recorded in the stage doc.

## 2. Execution schedule (with two agents)

```text
time ──────────────────────────────────────────────────────────────────────►
A:  [00 repo] [01 parser] [02 crawler] [03 builder] [04 coords] ....... [08 ops]
B:  [05 shell] ...... [06 filters] ┐
                       [07 map]     ┴─ parallel after 05 contract merges ─► [08 e2e/a11y]
C (reviewer): after 01, 03, 04, 06+07, 08 → run matrix, file defects
```

Ordering constraints (hard):

1. Stage 05's state contract must be **merged** before 06 and 07 start in parallel.
2. Stage 04 needs Stage 03's dataset shape (fields already present as `null`).
3. Stage 08 needs a real dataset from Stage 04 and a working map from 07.
4. Nobody starts a stage before its dependency's acceptance checklist is green.

## 3. Handoff artifacts between agents

| From → To | Artifact | Where |
|---|---|---|
| A → B | dataset JSON contract | `web/src/test/fixtures/dataset.sample.json` (committed, hand-written in 05, must match `02-architecture.md` §3.3) |
| A → B | real dataset for manual checks | `data/runs/<date>/dataset.json` (uncommitted; `make publish`) |
| B → C | e2e fixtures | `web/e2e/**` + `globalSetup` copy of the sample dataset |
| A → C | run manifest | `data/raw/<runId>/manifest.json` |

## 4. Prompt templates

Paste one per agent session. Each is self-contained: the agent has no memory of this conversation.

### 4.1 Agent A — Stages 00–01

```text
You are implementing part of a small scraping + map project in /home/jeka/DeepseekBench.

Read first, in this order:
  plan/README.md
  plan/01-discovery.md            (the site contract; selectors and quirks)
  plan/02-architecture.md §2-§4, §6
  plan/stage-00-bootstrap.md
  plan/stage-01-search-parser.md

Scope: Stage 00 then Stage 01, exactly as written. Files: Makefile, .gitignore, README.md,
pipeline/**. Do not touch web/** or plan/**.

Method: strict TDD. For every task: write the test listed in the stage doc, run it, confirm it fails
with a message that matches the doc, implement, run again, commit with the message given in the doc.
Never weaken an assertion to make it pass; if the doc is wrong, stop and report.

Fixtures already exist in plan/assets/fixtures/ — copy them, do not re-download. No network in tests.
Definition of done: `make check` green, stage acceptance checklist ticked, stage "Result" table filled
with real test counts, one commit per task.

Report back: commit hashes, test counts, anything you had to change in the plan and why.
```

### 4.2 Agent A — Stages 02–04

```text
Continue the pipeline in /home/jeka/DeepseekBench (repo already bootstrapped, Stage 01 merged).

Read first: plan/01-discovery.md §3-§8, plan/02-architecture.md §3, §6, then
plan/stage-02-crawler.md, stage-03-dataset-builder.md, stage-04-enrichment.md.

Scope: pipeline/** only. No network in unit tests: inject a FakeFetcher built from the fixtures.
The single opt-in network test is Stage 02 task 2.5.

Hard rules:
- sequential requests, delay floor 1.0 s, one detail-page fetch per building, cached across runs;
- only state == available reaches the dataset;
- deterministic output; atomic writes (.tmp + os.replace); resumable crawl.

Before any real crawl, ask: a live run hits a third-party site.

Definition of done: all stage acceptance checklists green, `pytest pipeline -q` ≥ 90 tests,
`connex-watch crawl --preset dev --dry-run` prints exactly 2 URLs.

Report back: commit hashes, test counts, the measured cold-run request count and wall time.
```

### 4.3 Agent B — Stages 05–07

```text
You are building the viewer for a small internal tool in /home/jeka/DeepseekBench.

Read first: plan/02-architecture.md §3.3, §4, §5 (UI spec and behaviour rules F1-F11),
plan/stage-05-web-shell.md, stage-06-filters.md, stage-07-map.md.

Scope: web/** only. Stage 05 first; stop after it is green and report, because 06 and 07 will be
split across two parallel sessions afterwards.

Rules:
- pure logic in src/lib/**, components dumb, one data flow in App.tsx;
- every rule F1-F11 has a test; Leaflet is mocked in component tests;
- pnpm, React 18, react-leaflet 4.2.1, Tailwind 3.4, CARTO Positron tiles with OSM+CARTO attribution;
- links: target="_blank" rel="noopener noreferrer";
- no `any`, no new dependency without a one-line justification in the commit body.

Definition of done per stage: `pnpm -C web test`, `pnpm -C web typecheck`, and (for 07)
`pnpm -C web e2e` all green.

Report back: commit hashes, test counts, bundle size, and a screenshot path of the final UI.
```

### 4.4 Agent B — Stage 07 only (parallel with 06)

```text
Implement Stage 07 of the viewer in /home/jeka/DeepseekBench (Stage 05 merged, Stage 06 may be
in flight in another session).

Read: plan/02-architecture.md §5.2 (F6-F9), plan/stage-07-map.md.

Scope — create/modify ONLY:
  web/src/components/MapView.tsx, BuildingPanel.tsx, MapLegend.tsx
  web/src/App.tsx (only the map/selection wiring, no filter changes)
  web/playwright.config.ts, web/e2e/**
  web/src/test/setup.ts

Do not edit FilterBar.tsx, ResultsList.tsx, BuildingRow.tsx, lib/**, state/** — another session owns
them. If you need a change there, write it in your report instead of doing it.

Definition of done: MapView/MapEvents/BuildingPanel tests green, 7 Playwright scenarios green,
tile attribution asserted.

Report back: commit hash, test counts, the exact diff you needed outside your scope (if any).
```

### 4.5 Agent C — Reviewer (read-only)

```text
You are reviewing a stage in /home/jeka/DeepseekBench. You do not write code.

Inputs: the stage doc (plan/stage-NN-*.md), plan/90-test-matrix.md, the git diff of the stage branch.

Check, in this order:
1. `make check` is green; paste the last 5 lines.
2. Every acceptance checkbox in the stage doc is actually evidenced (run the command, do not trust
   the tick).
3. The stage's rows in 90-test-matrix.md exist and their tests fail when the implementation is
   reverted (spot-check one: `git stash` the file, run the test, expect red, restore).
4. No network in unit tests: `pytest -m "not network"` with Wi-Fi off, or grep for `httpx` outside
   fetch.py.
5. No weakened assertions: search the diff for `assert True`, `skip`, `xfail`, `toBeTruthy()` used
   where a value was expected.
6. Determinism: run the stage's build twice and diff the output (ignoring timestamps).
7. Commit hygiene: one commit per task, `stage-NN:` prefix, no unrelated files.

Output: a table `Finding | Severity (blocker/major/minor) | File:line | Suggested fix`. No code.
Blockers must be fixed before the stage is marked done.
```

---

## 5. Merge and commit protocol

1. Branch per stage: `stage-03-dataset-builder`.
2. Commits: one per task, message from the stage doc, body lists the tests added.
   Push the stage branch: `git push -u origin stage-NN`.
3. Before merge: `make check` + the stage's acceptance commands, pasted into the stage "Result" table
   in the same commit as the last task.
4. Merge with `--no-ff` so each stage is one visible unit in `git log --graph`, then
   `git push origin main` (SSH remote, owner's key).
5. Tag after Stage 04 (`pipeline-v1`) and after Stage 08 (`v1.0`).

## 6. What to do when the site changes

1. `pytest pipeline -m network` fails or `crawl` exits 4 → do **not** patch the parser blind.
2. Re-capture one page: `curl -sS --compressed "<search url>" -H "User-Agent: …" > /tmp/probe.html`.
3. Diff against `pipeline/tests/fixtures/search-dev-p1.html`; identify the changed selector.
   **Redact before committing** (Maps key, `csrf_token`, `tel:` links) — see
   `assets/fixtures/README.md`.
4. Update `01-discovery.md` §2 selector table **first**, then the fixture, then the parser, then the
   golden snapshot — in that order, one commit each.
5. Never update the snapshot without explaining the diff in the commit body.

## 7. Time budgets and stop rules

| Stage | Budget | If exceeded |
|---|---|---|
| 00 | 25 min | split task 0.2 into two commits |
| 01 | 60 min | the parser is over-fitting: cut optional fields (image, floor), keep required ones |
| 02 | 60 min | drop `--full` and `--resume` to Stage 04 follow-up, but keep checkpoints |
| 03 | 45 min | skip the `run-mini` merge cases, keep dedupe + grouping |
| 04 | 50 min | ship without `project_code` merging (title-only ids), note it in the dataset stats |
| 05 | 50 min | ship without `StatusBar` tooltips |
| 06 | 60 min | drop the `datalist` and the 200-building perf test |
| 07 | 60 min | drop the Playwright mobile scenario (Stage 08 has it) |
| 08 | 60 min | drop systemd units (cron line is enough) |

Cutting scope is allowed **only** by editing the stage doc in the same commit, so the plan never
lies about what exists.
