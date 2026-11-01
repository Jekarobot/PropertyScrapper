---
id: 04-01
stage: 04
title: Detail-page coordinates from the map literal
points: 2
owner: pipeline
status: todo
depends_on: [01-02, 00-04]
files:
  - pipeline/src/connex_watch/detail_parser.py
  - pipeline/tests/test_detail_parser.py
---

# 04-01 · Detail-page coordinates from the map literal

**Points:** 2 · **Stage:** 04 — Enrichment · **Owner lane:** pipeline · **Depends on:** 01-02, 00-04 · **Unblocks:** 04-02, 04-03, 04-04

## Goal

Create `DetailInfo` and `parse_detail_page(html)`, so one unit detail page yields the building's coordinates plus its canonical identity fields.

## Why this task exists

After stage 03 the dataset has one entry per building, but `lat`/`lng` are nullable and always null: the search card carries no position. The only place the site exposes coordinates is the unit detail page (~220 KB, with an embedded Google Maps literal). Without this parser nothing can be mapped, and every later task in the stage (cache, loop, merge, CLI) has nothing to store. It must read *one* specific literal — the page also binds a second map and a street-view panorama — reject nonsense, and never touch the Google key that sits in the same script block.

## Current state (preconditions)

- 01-02 done: `pipeline/src/connex_watch/models.py` exists, pydantic v2 is the model layer.
- 00-04 and 00-05 done: `pipeline/tests/fixtures/detail-cx84121.html` exists (222 117 B, redacted) and the conftest HTML loader works; `pytest pipeline -q` is green.
- `detail_parser.py` and `tests/test_detail_parser.py` do not exist.

## Read before you start

- `CLAUDE.md` §3 (TDD), §6 (samples untrusted), §10 (never store/log a detail page, never use the site's Google key).
- `plan/01-discovery.md` §7 — `center:` literal, `LatLng(...)` fallback, breadcrumb table, "do not infer state here", "coordinates are optional".
- `plan/02-architecture.md` §3.3 (nullable `lat`/`lng`) and §6 rule P6.
- `pipeline/tests/fixtures/detail-cx84121.html` — the one `center: {lat: …}` literal, the `LatLng(...)` marker call, the second map bound to a local `fenway` object, the canonical link.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `pipeline/src/connex_watch/detail_parser.py` | `DetailInfo`, coordinate extraction, bbox |
| create | `pipeline/tests/test_detail_parser.py` | the seven cases below |

Nothing else may change: no `__init__` export, no cache, no fetcher, no CLI.

## Test first (red)

Write these before the module. Synthetic inputs are derived **in the test** from the frozen fixture by string replacement — never commit a second HTML file, never re-capture a page.

1. `test_detail_parser.py::test_fixture_center_literal_yields_exact_coordinates` — frozen fixture → `lat == 13.7187616`, `lng == 100.5202356`, both real `float`.
2. `...::test_first_center_literal_wins` — a decoy literal (`center: {lat: 13.7, lng: 100.5}`) prepended to the page → `13.7` / `100.5`, not the marker's numbers.
3. `...::test_latlng_fallback_used_when_center_is_absent` — every `center:` renamed to `centre:` → still `13.7187616` / `100.5202356`, taken from `LatLng(…)`.
4. `...::test_coordinates_outside_thailand_are_dropped` — both literals rewritten to `51.5074` / `-0.1278` → `lat is None`, `lng is None`.
5. `...::test_bounding_box_edges_are_inclusive` — parameterised: `(5.5, 97.0)` and `(20.5, 106.0)` kept, `(5.49, 100.5)` and `(13.7, 96.99)` dropped; pins the exact constants.
6. `...::test_identity_fields_parsed_from_fixture` — `project_code == "p00348"`, `building_name == "The Bangkok Sathorn"`, `unit_code == "CX-84121"`, `reason is None`.
7. `...::test_second_map_block_does_not_change_the_result` — the second map is bound to a local object plus a `StreetViewPanorama`; the result stays `13.7187616` / `100.5202356`, never `0.0` / `0.0`.

Expected red: `ModuleNotFoundError: No module named 'connex_watch.detail_parser'`, then `ImportError: cannot import name 'DetailInfo'` once the module exists but is empty.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_detail_parser.py -q`

## Implement (green)

- `DetailInfo` — pydantic v2 model, **exactly** six optional fields in this order: `lat`, `lng` (float or None), `project_code`, `building_name`, `unit_code`, `reason` (str or None); all default None. No state field, no state helper (04-04 enforces that boundary).
- `parse_detail_page(html: str) -> DetailInfo` — pure: no network, no file IO, no HTML logging or retention; the input is text the caller already fetched.
- Coordinates, in order: (1) the **first** `center:` literal with numeric `lat`/`lng`; (2) only when no such literal exists anywhere, the first `LatLng(` pair. Scan the text with a targeted pattern — not a JSON/HTML parse, the literal is JavaScript inside HTML.
- Bounding box: named constants `THAILAND_LAT_MIN = 5.5`, `THAILAND_LAT_MAX = 20.5`, `THAILAND_LNG_MIN = 97.0`, `THAILAND_LNG_MAX = 106.0`, each commented with the country extent and why it exists (second map block, street-view defaults, wrong template). Bounds inclusive; a pair outside → both None. Never retry the other literal after a rejection: the fallback is for absence only.
- Identity, straightforward version: first `project_code=` inside `.map-nav`, the text of the `.map-nav` link carrying `project_code=`, the `CX-<digits>` token in the last `.map-nav` segment → `p00348`, `The Bangkok Sathorn`, `CX-84121`. 04-02 owns the strict rules and the reasons.
- `reason` stays None here. One-sentence docstring on the model and on the parser. No `from_html_file` helper, no CLI, no Google key read.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_detail_parser.py -q` — seven green.

## Refactor / keep it DRY

The two coordinate patterns and the box check exist once, as module-level items shared by parser and tests (a private `_in_thailand` reused by case 5 is fine). `.map-nav` scoping is one helper that 04-02/04-03 extend. Re-run the tests after any refactor.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_detail_parser.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 7 passed, whole suite green, `ruff` exits 0. Manual check: `grep -c "center:"` on the fixture prints 1; if it prints more, re-read this task rather than relax the assertion.

## Out of scope (do not do)

- The cache, the enrichment loop, the CLI — 04-05 onward. The reason vocabulary and the truncated-HTML case — 04-02, 04-03.
- Any change to `models.py`, `build_dataset.py`, the dataset schema, or the fixtures. No fetching: `detail_parser.py` must not import `httpx` or `fetch`. Price, size and availability come from the search card only.

## Definition of done

- [ ] All seven cases exist, named as written, and pass.
- [ ] Red run observed and one failure line quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] Only the two files above plus this task file are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
04-01: parse detail-page coordinates into DetailInfo

- points: 2
- red: test_fixture_center_literal_yields_exact_coordinates failed (ModuleNotFoundError: detail_parser)
- add DetailInfo and parse_detail_page with center/LatLng extraction
- reject coordinates outside the named Thailand bounding box
- suite: N passed
```

Files in the commit: the two files above and this task file (status + Result). Nothing else.

## Traps and review notes

- Never parse the page as JSON/YAML/HTML around the script: it is JavaScript in legacy markup, and a generic "any `lat`/`lng` pair" scan picks the wrong block. Do not fall back to `LatLng(...)` when the `center:` literal exists but is out of bounds — absence and rejection are different outcomes.
- Anchor the numeric pattern on both delimiters; a greedy pattern glues digits across `, lng:`.
- Never `print`, log or keep the HTML; tests must not assert on its byte size.
- Reviewers: no `AIza`/`key=` in the module, no `httpx` import, no state field, bbox edges inclusive.

## Plan samples are untrusted

The plan shows the field list and two patterns as illustration. Verify them against the frozen fixture and the installed pydantic version (`CLAUDE.md` §6); the fixture decides what matches. Record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
