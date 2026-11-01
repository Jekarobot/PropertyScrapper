---
id: 04-03
stage: 04
title: Coordinate failure reasons and truncation tolerance
points: 1
owner: pipeline
status: todo
depends_on: [04-01]
files:
  - pipeline/src/connex_watch/detail_parser.py
  - pipeline/tests/test_detail_parser.py
---

# 04-03 · Coordinate failure reasons and truncation tolerance

**Points:** 1 · **Stage:** 04 — Enrichment · **Owner lane:** pipeline · **Depends on:** 04-01 · **Unblocks:** nothing yet

## Goal

Give every coordinate failure a fixed reason string and make the parser tolerate a truncated or broken page instead of raising.

## Why this task exists

Coordinates are optional: a building without them stays in the dataset and is shown as "no location". That only works if "we could not get coordinates" is a **value**, not an exception and not a silent null — the enrichment loop caches the failure and needs to tell an out-of-bounds page (wrong map block, `0.0`/`0.0` street-view default) from a page that simply has no literal. The same reason string ends up in the cache entry and the run log, so it is part of the diagnosis when the map thins out after a site change. A single `None` cannot carry that distinction.

## Current state (preconditions)

- 04-01 and 04-02 done: `DetailInfo` with six fields, `parse_detail_page`, coordinate extraction, bounding box, scoped identity, and the identity reasons `no-breadcrumb` / `no-project-code` / `no-unit-code` all exist and are green (thirteen cases).
- `pipeline/tests/fixtures/detail-cx84121.html` unchanged: the map script sits far below the first 2 KB of the document.
- `pytest pipeline -q` is green.

## Read before you start

- `CLAUDE.md` §3 and §6.
- `plan/01-discovery.md` §7 — "Coordinate failures" and the two failure modes the check protects against.
- `plan/02-architecture.md` §3.3 — `lat`/`lng` may be null and the building is still listed.
- `plan/stage-04-enrichment.md` Task 4.1 cases 3, 4 and 6.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/detail_parser.py` | coordinate reason strings and precedence |
| modify | `pipeline/tests/test_detail_parser.py` | the six new cases, existing cases untouched |

Nothing else may change: no cache, no enrichment loop, no fixture edits, no logging setup.

## Test first (red)

1. `test_detail_parser.py::test_no_coordinate_literal_yields_none_and_reason` — the fixture with the `center:` literal and every `LatLng(...)` pair stripped → `lat is None`, `lng is None`, `reason == "no-coordinate-literal"`, identity still parsed (`p00348`).
2. `...::test_out_of_bounds_coordinates_yield_reason` — both literals rewritten to `0.0` / `0.0` → both coordinates None, `reason == "out-of-bounds"`.
3. `...::test_rejected_literal_does_not_fall_back_to_latlng` — the `center:` literal made out of bounds while `LatLng(13.7187616, 100.5202356)` stays valid → still None and `"out-of-bounds"`, so a rejected value is never rescued by the fallback.
4. `...::test_truncated_html_returns_all_none_without_exception` — the first 2 KB of the fixture (`html[:2048]`) → all six fields None, `reason == "no-coordinate-literal"`, and no exception raised.
5. `...::test_coordinate_reason_takes_precedence_over_identity_reason` — a page with neither a coordinate literal nor a `.map-nav` → `reason == "no-coordinate-literal"`, not `"no-breadcrumb"`.
6. `...::test_full_fixture_has_no_reason` — the untouched fixture → `reason is None`, guarding against a reason being set on a successful parse.

Expected red: `AssertionError: assert None == 'no-coordinate-literal'` for cases 1, 2 and 4 (04-01/04-02 never set a coordinate reason) and `assert 'no-breadcrumb' == 'no-coordinate-literal'` for case 5.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_detail_parser.py -q`

## Implement (green)

- Keep `parse_detail_page(html) -> DetailInfo` unchanged in shape. No new field, no new module, no cache.
- Reason vocabulary is closed and lives in module constants: coordinate defects `"no-coordinate-literal"` and `"out-of-bounds"`, identity defects `"no-breadcrumb"`, `"no-project-code"`, `"no-unit-code"`. No other string may be produced.
- Precedence: coordinate defects first (no literal, then out-of-bounds), then identity defects in the 04-02 order, then None. The first defect found is the reason; later defects are not appended.
- Failure is never an exception: a truncated document, a page with no script block, and a page with no breadcrumb all return a fully populated `DetailInfo` of Nones. Do not catch broad exceptions around the whole parse to achieve that — make each extraction return None when its target is absent.
- Do not log, print or store the HTML, and do not add a reason for a page that resolved everything.
- One-sentence docstring stating that the reason is the first defect in a fixed precedence and that no input raises.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_detail_parser.py -q` — nineteen green.

## Refactor / keep it DRY

The reason strings and the precedence order exist once: one helper that decides the reason from the already-computed fields, called last by the parser. Do not scatter `if ... reason =` assignments through each extraction branch, and do not duplicate the bounding check in the reason helper — ask the existing `_in_thailand`. Re-run after refactoring.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_detail_parser.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 19 passed, whole suite green, `ruff` exits 0. Manual check: `grep -c "LatLng("` on the fixture prints 1; the fallback and the rejected-literal cases both depend on that single occurrence.

## Out of scope (do not do)

- Retrying, caching or logging the failure — 04-05 onward owns the cache, the loop owns the logs.
- New reason strings for HTTP errors, timeouts or non-200 responses: the fetcher reports those, not the parser.
- Guessing coordinates from an address, a district centroid, or a geocoder. No geocoding at all.
- Changing the success path or the identity rules.

## Definition of done

- [ ] All six new cases exist, named as written, and pass; the thirteen earlier cases still pass.
- [ ] Red run observed and one failure line quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] Only the two files above plus this task file are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
04-03: add coordinate failure reasons to the detail parser

- points: 1
- red: test_no_coordinate_literal_yields_none_and_reason failed (assert None == 'no-coordinate-literal')
- add no-coordinate-literal and out-of-bounds reasons
- tolerate truncated HTML instead of raising
- suite: N passed
```

Files in the commit: the two files above and this task file (status + Result). Nothing else.

## Traps and review notes

- Do not swallow exceptions to get "no raise": a `try/except Exception` around the parser hides real bugs and turns a broken selector into a permanent `no-coordinate-literal`. Absence must be a branch, not a caught error.
- Case 3 is the subtle one: falling back to `LatLng` after rejecting a literal would publish coordinates the bounding box just refused.
- `reason` must not be `""`, `"unknown"` or a formatted sentence; downstream code compares the exact strings.
- Reviewers: `grep -n "except"` in the parser should show nothing; `grep -n "out-of-bounds"` should show a constant, not a literal at the call site.

## Plan samples are untrusted

The plan lists the two reason strings but no implementation. Confirm the strings here are the ones the cache format in `plan/stage-04-enrichment.md` Task 4.2 shows, and record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
