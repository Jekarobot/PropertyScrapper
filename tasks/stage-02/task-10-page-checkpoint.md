---
id: 02-10
stage: 02
title: Atomic page checkpoint round-trip
points: 1
owner: pipeline
status: todo
depends_on: [02-09]
files:
  - pipeline/src/connex_watch/store.py
  - pipeline/tests/test_store_resume.py
---

# 02-10 · Atomic page checkpoint round-trip

**Points:** 1 · **Stage:** 02 — Crawler · **Owner lane:** pipeline · **Depends on:** 02-09 · **Unblocks:** 02-11

## Goal
Save one parsed page as `pages/page-0001.json` with a single atomic write, in the documented raw shape, so a crash can never leave a half-written checkpoint and a later read can rebuild the page object.

## Why this task exists
The nightly crawl is ~75 requests and a cold run ~1100; a killed run must resume from the last complete page instead of starting over (`plan/02-architecture.md` §6 rule P10). That only works if a page file is either absent or complete, and if its JSON carries enough to rebuild the parsed page exactly. This is also the first raw-layer file stage 03 reads, so its field names become a cross-stage contract.

## Current state (preconditions)
- 02-09 is done: `RunStore(root, clock=...)`, `create(run_id, spec)` and the private path and atomic-write helpers exist; `test_store_resume.py` has four green cases.
- Stage 01 exposes `PageResult` (`cards`, `page_no`, `total_pages`, `total_units`, `is_error_stub`, with camelCase aliases) and `state_histogram(states)` in `search_parser.py` (01-02, 01-06); `build_search_url(spec, page_no=n)` is available.
- No page file has ever been written; `has_page`/`load_page` do not exist (02-11).

## Read before you start
- `CLAUDE.md` §3 (determinism, injected clock), §5 (one write helper), §6, §11 (never commit `data/`).
- `plan/02-architecture.md` §3.2 — the exact page JSON keys and the atomic write rule.
- `plan/stage-02-crawler.md` Task 2.3 cases 2 and 3, and the "never write partial JSON" trap; `plan/90-test-matrix.md` row P-12.
- `plan/assets/fixtures/README.md` — the card facts used below (16 cards, `CX-84121`, ฿45 000, 65 sqm, floor 27).

## Files to touch
| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/store.py` | `save_page` and the page ↔ JSON mapping |
| modify | `pipeline/tests/test_store_resume.py` | the four cases below |

Nothing else: no `models.py` change, no `load_page` yet, no manifest change.

## Test first (red)
Parse `search_dev_p1`, create the store with a frozen clock and the DEV spec, and save page 1.

1. `test_store_resume.py::test_save_page_writes_page_0001_json` — `<root>/run-a/pages/page-0001.json` exists (four-digit padding) and `json.loads` gives `schemaVersion == 1`, `pageNo == 1`, `totalPages == 2`, `totalUnits == 32`, `fetchedAt` equal to the frozen clock as `YYYY-MM-DDTHH:MM:SSZ`, `url` equal to the DEV canonical page-1 URL, `stateCounts == {available: 4, check_availability: 5, not_available: 7}`, `len(cards) == 16`, and `cards[0]` with `id == "CX-84121"`, `state == "available"`, `priceThb == 45000`, `bedrooms == 1`, `bathrooms == 1`, `floor == 27`, `sizeSqm == 65`, `position == 1`.
2. `test_store_resume.py::test_save_page_leaves_no_tmp_file` — after the save (and after two saves in a row) `pages/` contains exactly one file and no name ends in `.tmp`; a leftover temp file would make 02-11 skip a page that is not there.
3. `test_store_resume.py::test_saved_page_round_trips_through_page_result` — rebuild a `PageResult` from the JSON through the model's validation entry point (the model already accepts its camelCase aliases) and assert equality with the saved page: page number, totals, stub flag, and a card-for-card comparison of ids, states, prices, sizes and positions.
4. `test_store_resume.py::test_save_page_twice_overwrites_and_never_duplicates` — save page 1 twice → still exactly one file for page 1, content equal to the second write, and no `page-0001-1.json` variant; saving is idempotent per page number.

Expected red symptom: `AttributeError: 'RunStore' object has no attribute 'save_page'`. A typo failure is not red.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_store_resume.py -q`.

## Implement (green)
- `save_page(page)` — write `<root>/<runId>/pages/page-<pageNo:04d>.json` and return nothing. Serialise with `json.dumps` (UTF-8, `indent=2`, `ensure_ascii=False`, trailing newline) into `<name>.tmp` in the same directory, close it, then `os.replace` onto the final name; the temp file never survives a normal call.
- Payload keys in documented order: `schemaVersion` (`1`), `pageNo`, `totalPages`, `totalUnits`, `fetchedAt`, `url`, `stateCounts`, `cards`. `fetchedAt` comes from the store's injected clock; `url` is `build_search_url(spec, page_no=<pageNo>)` — the URL the crawler requested. `save_page` takes no URL and no timestamp argument.
- `stateCounts` is composed in the store from the page's card states with the stage-01 `state_histogram` helper (01-06) — it is not a model field, and it always has all three keys. `cards` keep the raw file's field names, page order and positions.
- Serialisation uses the model's declared camelCase aliases and adds the raw-only keys (`schemaVersion`, `fetchedAt`, `url`, `stateCounts`) in one pair of functions in `store.py`; 02-11 imports the reader, nobody duplicates it.
- No manifest update, no logging, no directory caching here.
- One-sentence docstring on `save_page` stating the path rule, the atomicity and the payload.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_store_resume.py -q` — eight green.

## Refactor / keep it DRY
`save_page` and the manifest writer share 02-09's atomic-write helper; the page mapping exists once, in the pair of functions 02-11 reuses. No second serialiser for tests. Re-run the suite.

## Verification (how to test it)
```text
. .venv/bin/activate && pytest pipeline/tests/test_store_resume.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 8 passed, the pipeline suite green, `ruff` exits 0. Manual check: `git status --short` shows no `data/` entry — everything was written under `tmp_path`.

## Out of scope (do not do)
- `has_page` / `load_page` / resume — 02-11. `finish`, counters, the spec guard — 02-12.
- Writing the stub page or a zero-card page: 02-07 decides they never become pages.
- A per-page checksum, page index, compression, or schema validation at write time.
- Saving raw HTML — only the parsed page (`CLAUDE.md` §10: parse, extract, discard).
- Updating `pagesFetched` or any other counter — 02-12 owns them.

## Definition of done
- [ ] All four cases exist, named as written, and pass; the 02-09 cases still pass.
- [ ] Red run observed and quoted in the commit body.
- [ ] Target tests + `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `store.py`, `test_store_resume.py` and this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit
```text
02-10: save page checkpoints atomically

- points: 1
- red: test_save_page_writes_page_0001_json failed (AttributeError: 'RunStore' object has no attribute 'save_page')
- add save_page with .tmp plus os.replace and the documented raw page payload
- keep one page-to-JSON mapping that 02-11 can read back
- suite: 57 passed
```

Files in the commit: `pipeline/src/connex_watch/store.py`, `pipeline/tests/test_store_resume.py`, and this task file — nothing else.

## Traps and review notes
- Do not write the final file directly: a crash mid-`json.dumps` leaves a page that parses as JSON and silently lost cards.
- Do not put the `.tmp` file outside `pages/`; `os.replace` across filesystems is not atomic.
- Do not name the file `page-1.json` or `page-00001.json`: 02-11's existence check and the plan's examples assume four digits.
- Do not re-derive `url` or `fetchedAt` at load time; they are written once and read back.
- Reviewer: case 3 must compare cards field by field, not only counts — a mapping that drops `position` passes a count assertion.

## Plan samples are untrusted
`plan/02-architecture.md` §3.2 shows a sample page document; this task pins the same keys, but the repo's `PageResult` field names win. Check the model before coding and record any field the model does not carry under `## Decisions`.

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
