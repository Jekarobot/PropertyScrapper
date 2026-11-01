---
id: 08-12
stage: 08
title: Full regression and release sign-off
points: 3
owner: any
status: todo
depends_on: [08-06, 08-09]
files:
  - tasks/stage-08/task-12-release-regression.md
---

# 08-12 · Full regression and release sign-off

**Points:** 3 · **Stage:** 08 — Hardening and release · **Owner lane:** any (release) · **Depends on:** 08-06, 08-09 · **Unblocks:** the release itself

## Goal

Run the whole test matrix against the finished repository, compare the live dataset with the 2026-11-01 baseline, investigate anything outside tolerance, and record the release decision with evidence in this file.

## Why this task exists

Everything so far was proved in pieces: unit suites offline, one live smoke test, one nightly against a stub, one mobile spec. Nobody has run the pipeline against the real site and the browser against the real dataset on the same day, and that is where the expensive failures live — a dataset 70 % coordinate-less, an e2e suite green on the sample and red on 1 100 units. It is 3 points because it is investigation before reporting.

## Current state (preconditions)

- 08-06 is done (`make check` includes the dataset step and passes without one) and 08-09 is done (20 e2e tests, 375 px included); stage 08 is merged on `main`, `.venv` is installed and `pnpm` is available.
- The owner has been asked for permission to run the live nightly; without it this task stops after the offline steps and reports `blocked`.
- Local `data/` may be stale, which is part of what is measured; fixtures are untouched.

## Read before you start

- `CLAUDE.md` §2 (bookkeeping in the same commit), §3, §8, §10 (live crawl only with explicit approval; never enable the timer).
- `plan/90-test-matrix.md` — every row, the release checklist and the anti-pattern table; `plan/stage-08-hardening-ops.md` Task 8.6 and the baseline numbers; `plan/02-architecture.md` §3.3, §6 P9, §7; `plan/stage-04-enrichment.md` Traps.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `tasks/stage-08/task-12-release-regression.md` | `## Result` filled with the evidence and the checklist |

Nothing else. No product code, no test change, no README, no unit file, no `Makefile`, no committed dataset. A defect found here becomes a **new**, owner-approved task, not part of this commit.

## Test first (red)

This task adds no test: its "red" is the state of the matrix before the investigation, quoted so that "it was green" is evidence, not memory.

1. `R1` `make check` — offline; expect exit 0, every lane green, the dataset step validating the file or printing its skip message; record the pytest and web counts and that line verbatim.
2. `R2` `pytest pipeline -m network -q` — the single opt-in live smoke test; expect `1 passed`. A failure means the site contract changed and everything downstream is suspect.
3. `R3` `make nightly` — **only with the owner's explicit approval in this session.** Expect exit 0, a warm-cache runtime under 5 minutes (under 80 requests) and a fresh `data/runs/<date>/dataset.json` published to `web/public/data/dataset.json`; without approval, stop, mark `blocked`, keep the offline evidence.
4. `R4` `pnpm -C web e2e` — expect 20 passed (15 desktop + 5 mobile) against the freshly published dataset, console collector silent.
5. `R5` size and `stats` — `du -h web/public/data/dataset.json` under 2 MB (more means a field duplicated per unit); write down `units`, `buildings`, `buildingsWithCoords`, `pagesScanned`, `partial`, `generatedAt`, `duplicatesDropped`, `excluded`, `droppedInvalid`.
6. `R6` baseline comparison — units ≈ 1 100, buildings ≈ 950, `buildingsWithCoords / buildings` ≥ 0.85, `pagesScanned` ≈ 73; compute the ratio and the deltas in the table. Do not round a ratio down to "fine".
7. `R7` coordinate investigation — **only if the ratio is below 0.85.** Count the buildings without coordinates, group by district, check each `coordSource` for `detail:`, inspect the cache for negatives, open one affected detail page by hand (owner-approved) for the `center:` literal.
8. `R8` hygiene and acceptance — clean `git status --short`, one commit per task, `git ls-files data web/public/data logs` empty, every stage-08 task `status: done`; then walk the checklist below, one evidence line each.

Expected red: before the run there is no fresh dataset, so `make check` prints its skip line and R5–R7 have nothing to measure; R2 may fail if the site changed since 2026-11-01. Quote the skip line and any failure as the starting evidence.

## Implement (green)

There is no code to write. Run the eight steps, keep every raw output, fill `## Result`, then apply the decision rule:

- All of R1–R8 green with the ratio at or above 0.85 → the release is approved; `Result` says so in one sentence with the four headline numbers and the commit. Anything red outside tolerance → `blocked` with the failing step, the counts that prove it and a new owner-approved task: a release task that edits code hides what shipped.
- A ratio below 0.85 with the detail pages still carrying `center:` is a cache or merge problem, not a site change: say which and block.
- Never enable, start or reload a systemd unit or cron entry, and never publish a dataset the owner has not seen.

## Refactor / keep it DRY

Nothing is implemented, so nothing is duplicated. Evidence is copied once from the command's own output, never retyped from memory.

## Verification (how to test it)

```text
make check
pytest pipeline -m network -q
make nightly
```

Expected: `check` exits 0; the network test is `1 passed`; `nightly` exits 0 within 5 minutes (owner-approved); e2e is 20 passed from R4. The `pytest -m network` step belongs to this task only.

## Out of scope (do not do)

- Fixing any defect found, changing a threshold, or re-running a failing step until it passes without understanding why.
- Enabling the timer, adding a cron entry, running the nightly twice "to compare", or leaving a background job running.
- Publishing outside the nightly's own atomic publish, editing the README, or committing `data/`, `web/public/data/`, `logs/` or `.probe/`.

## Definition of done

- [ ] R1–R8 executed, every output quoted in `## Result`; baseline and ratio recorded, with a decision for anything out of tolerance.
- [ ] The manual acceptance checklist is filled in, one evidence line per item.
- [ ] `git status --short` shows only this task file; frontmatter `status: done` (or `blocked`); `## Result` complete.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
08-12: run the release regression and record the sign-off

- points: 3
- red: make check printed the dataset skip line; no fresh dataset for R5-R7
- run the matrix and record the baseline comparison, ratio and checklist
- suite: 20 e2e, network 1 passed, units=<n> buildings=<n> coords=<pct>
```

Files in the commit: `tasks/stage-08/task-12-release-regression.md` and nothing else; one commit, on trunk `main`.

## Traps and review notes

- Running `make nightly` without asking the owner is the unattended live crawl `CLAUDE.md` §10 forbids; approval is per session and not implied by this file.
- A warm-cache nightly and a cold one differ by twenty minutes and a thousand requests: say which produced the runtime you recorded.
- `buildingsWithCoords` can be high while the map is empty (buildings outside the default view), so the ratio alone does not close W-18; a green e2e run against `dataset.sample.json` proves nothing about the real data. Reviewers: `Result` holds raw output, the date, the commit hash and the four numbers; `git log --oneline` shows one commit per task.

## Plan samples are untrusted

Task 8.6 lists the commands and the baseline, but its `make nightly` and dataset paths predate 08-02/08-06's targets. Verify each command against the shipped `Makefile` and `cli.py`, keep the baseline numbers as stated, and record deviations under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |

### Matrix evidence

| Step | Command | Observed | Verdict |
|---|---|---|---|
| R1 | `make check` | | |
| R2 | `pytest pipeline -m network -q` | | |
| R3 | `make nightly` (owner-approved) | | |
| R4 | `pnpm -C web e2e` | | |
| R5 | dataset size and `stats` | | |
| R6 | baseline comparison | | |
| R7–R8 | investigation, hygiene and checklist | | |

### Baseline comparison (2026-11-01)

| Metric | Baseline | Measured | Delta | Within tolerance |
|---|---|---|---|---|
| units | ≈ 1 100 | | | |
| buildings | ≈ 950 | | | |
| buildingsWithCoords | ≥ 85 % | | | |
| pagesScanned | ≈ 73 | | | |

### Manual acceptance

- [ ] `make nightly` on a warm cache finishes in under 5 minutes
- [ ] dataset `stats.partial == false`
- [ ] `buildingsWithCoords / buildings >= 0.85`
- [ ] three rows and at least two bubbles in the default view; drag updates the list
- [ ] clicking a bubble fills the panel; "view ↗" opens the real site in a new tab
- [ ] 375 px: filter sheet works, map toggles, no horizontal scroll
- [ ] `/data/dataset.json` is under 2 MB
