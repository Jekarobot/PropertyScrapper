---
id: 02-09
stage: 02
title: Run directory, manifest creation
points: 1
owner: pipeline
status: todo
depends_on: [02-04]
files:
  - pipeline/src/connex_watch/store.py
  - pipeline/tests/test_store_resume.py
---

# 02-09 · Run directory, manifest creation

**Points:** 1 · **Stage:** 02 — Crawler · **Owner lane:** pipeline · **Depends on:** 02-04 · **Unblocks:** 02-10, 02-12, 02-13

## Goal
Create one run directory per crawl — `data/raw/<runId>/` with a `pages/` subdirectory and a `manifest.json` recording the run id, the search spec and a null stop reason — and refuse to create a run id that already exists.

## Why this task exists
The raw layer is the only durable output of the crawl, and every later stage reads it: the dataset builder (03-01), the enrichment pass (04-09) and the nightly script (08-02) find a run by its directory and learn what it was from its manifest. Without a fixed layout and a manifest written at start, a crashed run is indistinguishable from a run that found nothing, and `--resume` (02-11) cannot know whether an existing directory belongs to the search it is asked to continue.

## Current state (preconditions)
- 02-04 is done: `models.py` exports `StopPolicy` and `CrawlResult`; stage 01 exports `SearchSpec` with `to_url(page_no=n)`.
- `pipeline/src/connex_watch/store.py` and `pipeline/tests/test_store_resume.py` do **not** exist.
- `data/` is gitignored except `data/.gitkeep` (00-01); tests write under `tmp_path`, never into the repository.
- Nothing else in the package writes files; the fixture loaders are read-only.

## Read before you start
- `CLAUDE.md` §3 (inject the clock, byte-identical output), §5 (YAGNI: no database, one file per page), §6, §11 (never commit `data/`).
- `plan/02-architecture.md` §3.2 — the run layout and the manifest field list; §2 — the directory tree.
- `plan/stage-02-crawler.md` Task 2.3 cases 1 and 7; `plan/90-test-matrix.md` row P-12.
- `plan/01-discovery.md` §4 — canonical parameter order, so the stored spec URL is diff-friendly.

## Files to touch
| Action | Path | What goes there |
|---|---|---|
| create | `pipeline/src/connex_watch/store.py` | `RunStore`, the layout, the manifest |
| create | `pipeline/tests/test_store_resume.py` | the four cases below |

Nothing else: no `crawl.py`, no `cli.py`, no `pyproject.toml`, and no file under `data/`.

## Test first (red)
Every case uses `tmp_path` as the raw root and a frozen clock (an injected callable returning a fixed timezone-aware UTC datetime), so timestamps are exact strings, not "roughly now".

1. `test_store_resume.py::test_create_makes_the_run_layout` — `RunStore(tmp_path).create("run-a", spec)` → `<tmp_path>/run-a/pages` is a directory, `<tmp_path>/run-a/manifest.json` is a file, and the run directory contains exactly those two entries.
2. `test_store_resume.py::test_manifest_records_the_started_run` — the same call, then `json.loads` on the manifest: `runId == "run-a"`, `finishedAt is None`, `stopReason is None`, `pagesFetched == 0`, `requestCount == 0`, `errorCount == 0`, `notes == []`; `spec` holds exactly the ten canonical search parameters in the order of `plan/01-discovery.md` §4 (no page number), and `build_search_url(SearchSpec(**manifest["spec"]), page_no=1) == build_search_url(spec, page_no=1)` — the stored block reconstructs the same search.
3. `test_store_resume.py::test_started_at_is_iso_utc_with_a_z_suffix` — `startedAt` equals the frozen clock rendered as `YYYY-MM-DDTHH:MM:SSZ`, parses back with `datetime.fromisoformat` after replacing the `Z`, and is timezone-aware UTC; no offsets, no microseconds.
4. `test_store_resume.py::test_create_refuses_an_existing_run_id` — create `run-a`, write a marker file by hand inside `pages/`, then `create("run-a", spec)` again → `ValueError` mentioning `run-a`; the marker still exists and the manifest still shows the first `startedAt`. A second create never erases checkpoints.

Expected red symptom: `ModuleNotFoundError: No module named 'connex_watch.store'`. A typo failure is not red.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_store_resume.py -q`.

## Implement (green)
`store.py` exposes one public class, `RunStore`:

- `RunStore(root, *, clock=None)` — `root` is the raw base directory (the CLI passes `--out`, default `data/raw`); `clock` returns an aware UTC datetime and defaults to the real UTC clock. Tests inject it; no function under test calls `datetime.now()`.
- `create(run_id, spec)` — make `<root>/<run_id>/pages/`, then write `manifest.json` with exactly: `runId`, `startedAt` (from the clock, formatted above), `finishedAt: None`, `spec` (the `SearchSpec` parameter block: `project_code`, `listingtype`, `propertytype`, `bedroom`, `minsize`, `maxsize`, `minprice`, `maxprice`, `special`, `sort`, values as the model dumps them and no page number), `pagesFetched: 0`, `stopReason: None`, `requestCount: 0`, `errorCount: 0`, `notes: []` — in that order. Create the root if missing, and return the store bound to that run so 02-10 can save pages on it.
- An existing run id → `ValueError` naming it, checked before anything is written.
- UTF-8, `ensure_ascii=False`, `indent=2`, trailing newline; the write is atomic (`.tmp` then `os.replace`), so a half-written manifest is impossible.
- Path helpers (run dir, pages dir, manifest path) are private and used by 02-10/02-12; paths are never built inline twice.
- One-sentence docstring on the class and on `create` stating the layout, the injected clock and the `ValueError`.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_store_resume.py -q` — four green.

## Refactor / keep it DRY
The timestamp format, the atomic write and the path helpers each exist once; 02-10 and 02-12 reuse them instead of re-implementing `.tmp` + `os.replace`. Re-run the suite after refactoring.

## Verification (how to test it)
```text
. .venv/bin/activate && pytest pipeline/tests/test_store_resume.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 4 passed, the pipeline suite green, `ruff` exits 0. Manual check: after the suite, `git status --short` shows no `data/` entry — all files were written under `tmp_path`.

## Out of scope (do not do)
- Writing page files, `has_page`/`load_page` — 02-10 / 02-11. `finish()` — 02-12.
- Resume, spec-mismatch detection, `record_request` — 02-11 / 02-12.
- Pruning, retention, a runs index, a lock file, a database (explicit non-goal).
- Schema validation of the manifest, or choosing the run id from a timestamp — 02-13 does that at the CLI boundary.

## Definition of done
- [ ] All four cases exist, named as written, and pass.
- [ ] Red run observed and quoted in the commit body (`ModuleNotFoundError` line).
- [ ] Target tests + `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `store.py`, `test_store_resume.py` and this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit
```text
02-09: create the run directory and manifest

- points: 1
- red: test_create_makes_the_run_layout failed (ModuleNotFoundError: No module named 'connex_watch.store')
- add RunStore with data/raw/<runId>/{manifest.json,pages/}
- write the manifest atomically with an injected clock and refuse an existing run id
- suite: 53 passed
```

Files in the commit: `pipeline/src/connex_watch/store.py`, `pipeline/tests/test_store_resume.py`, and this task file — nothing else.

## Traps and review notes
- Do not call `datetime.now()` inside `create`; the frozen clock is the only source of `startedAt`, and case 3 compares the exact string.
- Do not create the directory and then raise on the exists check — check first, so a failed create leaves no half-made run.
- Do not store the spec as a bare string, as the URL the operator typed, or as a wrapper object: store the parameter block, because 03-01/03-11 read it as the dataset's `search` block. A reordered URL must still produce the same block.
- Do not write with `sort_keys=True`: the documented field order is the runbook's reading order.
- Reviewer greps `store.py` for `open(` with `"w"`: every write goes through the one atomic helper.

## Plan samples are untrusted
`plan/02-architecture.md` §3.2 lists the manifest fields but not the `spec` shape; this task pins it to the parameter block used by 03-01's hand-written run fixture. Check the repo's `SearchSpec` before coding and record a deviation under `## Decisions` if the field list disagrees.

## Decisions
<!-- append during implementation; leave empty at planning time -->

## Result
| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
