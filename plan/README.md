# Connex Watch — implementation plan (docs only, no code)

A two-part project:

1. **Pipeline** (Python, runs on a schedule): crawl `connex.in.th` search results, keep only
   units whose state is `Available`, store raw JSON, then group units by condominium building and
   emit a small dataset (`dataset.json`) with price, size, district, building, coordinates and a
   link to each unit.
2. **Viewer** (React + Vite + Leaflet, static SPA): filter bar (size, bedrooms, district, building),
   map of buildings with bubbles at the bottom, list on the right, units of the selected building
   on the left.

This folder contains **only plans and research**. No implementation code. Execution happens later,
stage by stage, one commit per task.

---

## 0. What was verified against the live site (2026-11-01)

| Question | Answer | Evidence |
|---|---|---|
| Is there an API? | No. Server-rendered PHP + jQuery. No `fetch`/`$.ajax`/JSON endpoint on search or detail pages. | `plan/01-discovery.md` §2 |
| How many results per page? | 16 cards | parity of `div.col-xl-3…` blocks |
| Availability in markup? | Three CSS classes on the card: `available`, `not-available`, `check-availability` | `plan/01-discovery.md` §3 |
| Can we filter by availability server-side? | **No.** `status=1` is a decorative hidden field; GET and POST ignore it | `plan/01-discovery.md` §5 |
| Does `sort=` (empty) put available first? | Partially — available units cluster at the front, but the ordering is "recommended", not a filter. A project with 32 units has available/check/not mixed on page 1. | `plan/01-discovery.md` §6 |
| How big is the full run? | `8734` units / **546 pages** for the given filter, of which the available cluster is **~71 pages ≈ 1132 units** | `plan/01-discovery.md` §6 |
| Where are coordinates? | Only on the unit detail page, inside the Google Maps `initMap()` literal. **Same coordinates for all units of one building.** | `plan/01-discovery.md` §7 |
| Cost of one full run | ~75 search pages (~1.3 MB) + ~1000 detail pages (~24 MB), ~20–25 min at 1.2 s/request. Cached on later runs: ~2–3 min. | `plan/01-discovery.md` §8 |
| robots.txt | `Allow: /`, `Disallow: /backend` — crawling search pages is allowed | `plan/01-discovery.md` §9 |

These numbers are expected to drift. Plans never hardcode them as assertions on the live site;
they are baked into **fixtures** so tests stay deterministic.

---

## 1. Stage map

Every stage is a self-contained, commit-able unit. A stage ends with a green test suite and a
recorded result line in its own "Result" section (filled by the implementing agent).

| # | Stage | Depends on | Est. | Can run in parallel with |
|---|---|---|---|---|
| 00 | [Bootstrap](stage-00-bootstrap.md) | — | 25 min | — |
| 01 | [Search-page parser](stage-01-search-parser.md) | 00 | 60 min | 05 |
| 02 | [Crawler + checkpoints](stage-02-crawler.md) | 01 | 60 min | 03, 05 |
| 03 | [Dataset builder (grouping)](stage-03-dataset-builder.md) | 01 | 45 min | 02, 05 |
| 04 | [Building enrichment (coords)](stage-04-enrichment.md) | 03 | 50 min | 05, 06 |
| 05 | [Web shell + data layer](stage-05-web-shell.md) | 00 | 50 min | 01–04 |
| 06 | [Filters + results list](stage-06-filters.md) | 05 | 60 min | 07 |
| 07 | [Map + viewport sync](stage-07-map.md) | 05 | 60 min | 06 |
| 08 | [Hardening, E2E, ops](stage-08-hardening-ops.md) | 02, 04, 06, 07 | 60 min | — |

Supporting documents:

- [`01-discovery.md`](01-discovery.md) — the site contract: URLs, DOM selectors, quirks, measured
  behaviour, risk list. **Read this before writing any parser.**
- [`02-architecture.md`](02-architecture.md) — repo layout, data model + JSON schemas, stack
  choices with rejected alternatives, UI behaviour spec, layout wireframe.
- [`90-test-matrix.md`](90-test-matrix.md) — requirement → test level → file → command. Use it to
  prove a stage is done.
- [`91-agent-briefs.md`](91-agent-briefs.md) — copy-paste prompts per stage, file-ownership matrix,
  rules for parallel sub-agents.
- [`assets/fixtures/`](assets/fixtures/) — real HTML captured 2026-11-01, to be copied into
  `pipeline/tests/fixtures/` in Stage 00. **Tests must never hit the network.**

---

## 2. Global conventions

### Commit format

```
stage-03: group units into buildings

- add build_dataset() with title normalisation and district parsing
- 12 new tests, 31 total, all green
```

Prefix `stage-NN:` on the first line. One stage = one branch = N commits (one per task).
Never mix `pipeline/` and `web/` changes in one commit.

### Repository, branches, remote

- `origin` = `git@github.com:Jekarobot/PropertyScrapper.git` (SSH, owner's key). `main` holds the plan.
- One branch per stage: `stage-00` … `stage-08`. Never commit a stage branch straight onto `main`;
  merge with `--no-ff` after the stage checklist is green, then `git push`.
- `git push -u origin <stage-branch>` at the end of the stage; merge/push to `main` is the owner's
  decision (PR or local merge).
- Never force-push `main`, never amend a commit that has been pushed.

### Definition of done (per task)

1. A failing test exists **before** the implementation and was seen failing (paste the failure
   line into the commit body or the stage "Result" table).
2. Test passes; the whole suite passes; `ruff` and `tsc --noEmit` are clean.
3. No network in unit tests (`pytest -m "not network"` must pass offline).
4. Public functions have a docstring stating the contract in one sentence.
5. Stage doc updated: task checkboxes ticked, "Result" row filled.

### Rules that prevent rework

- **The pipeline never trusts the live site's counts.** `Of N` is a hint, the card list is the
  truth, and a page with zero cards ends the walk.
- **Fixtures are the contract.** If the site changes, one failing fixture test tells you exactly
  which selector broke.
- **Coordinates are per building, not per unit.** Never fetch a detail page per unit.
- **The dataset is the only interface between pipeline and web.** The web app knows nothing about
  `connex.in.th` HTML.
- **YAGNI guard:** no database, no auth, no server, no framework beyond Vite+React, no cross-run
  diffing in v1 (documented as backlog in `02-architecture.md` §9).

---

## 3. How to execute a stage

```bash
cd /home/jeka/DeepseekBench            # repo root after Stage 00
git switch -c stage-03
# 1. read the stage doc end to end, then plan/02-architecture.md §4
# 2. for each task: write the test, run it, watch it fail, implement, watch it pass
make test            # pytest + vitest
make check           # ruff + tsc + pytest + vitest
git commit -m "stage-03: …"
```

If a stage cannot be finished: commit the failing test **as `xfail` with a TODO comment**, write
the blocker in the stage doc, and stop. Never leave a red suite on `main`.

---

## 4. Sub-agent split (implementation phase)

Full briefs in [`91-agent-briefs.md`](91-agent-briefs.md). Summary:

| Agent | Owns (files) | Stages | Notes |
|---|---|---|---|
| **A — pipeline** | `pipeline/**` | 00 → 01 → 02 → 03 → 04 (sequential) | One agent, no parallel writers inside `pipeline/` |
| **B — web** | `web/**` | 05 → 06 and 07 in parallel | 06 and 07 are safe in parallel only after 05's state contract is merged |
| **C — reviewer** | nothing (read-only) | after each stage | Runs `make check`, re-runs the stage's acceptance checklist, files defects |

Stage 08 belongs to whoever is free; it touches `Makefile`, `scripts/`, `web/e2e/` and docs only.

---

## 5. Deliverables of this plan folder

```
plan/
  README.md             ← you are here
  01-discovery.md       site contract + evidence
  02-architecture.md    repo, data model, stack, UI spec
  stage-00..08-*.md     bite-sized TDD tasks
  90-test-matrix.md     proof-of-done matrix
  91-agent-briefs.md    sub-agent prompts + ownership
  assets/fixtures/*.html
```

Nothing in this folder is imported by the product. It is documentation and test data.
