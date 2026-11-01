---
id: 00-04
stage: 00
title: Copy frozen HTML fixtures and provenance README
points: 0.5
owner: pipeline
status: todo
depends_on: [00-03]
files:
  - pipeline/tests/fixtures/search-dev-p1.html
  - pipeline/tests/fixtures/search-dev-p2.html
  - pipeline/tests/fixtures/search-boundary-p71.html
  - pipeline/tests/fixtures/search-error-out-of-range.html
  - pipeline/tests/fixtures/detail-cx84121.html
  - pipeline/tests/fixtures/README.md
  - pipeline/tests/test_fixtures_provenance.py
---

# 00-04 · Copy frozen HTML fixtures and provenance README

**Points:** 0.5 · **Stage:** 00 — Bootstrap · **Owner lane:** pipeline · **Depends on:** 00-03 · **Unblocks:** 00-05, 04-01

## Goal

Copy the five frozen HTML captures into `pipeline/tests/fixtures/` with a provenance README, so parser tests stay offline.

## Why this task exists

Stage 01 parses cards, pagers and Buddhist dates out of this markup and stage 04 reads coordinates from the detail page.
Without committed copies every test needs the live site; without provenance nobody can tell which URL, status and
redactions the bytes came from, and the next "fix" is a re-capture. This task copies and documents only — no parser.

## Current state (preconditions)

- 00-03 is done: `pytest pipeline -q` reports 4 passed, `ruff` is clean, `.venv` exists. `pipeline/tests/fixtures/` does
  not exist; `plan/assets/fixtures/` holds the five files and their README.

## Read before you start

- `plan/assets/fixtures/README.md` — five files, URLs, HTTP 200s, raw sizes, redactions, frozen rule.
- `plan/01-discovery.md` §2–§3, §7; `CLAUDE.md` §3, §10, §11; `plan/02-architecture.md` §2.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `pipeline/tests/fixtures/search-dev-p1.html`, `search-dev-p2.html`, `search-boundary-p71.html`, `search-error-out-of-range.html`, `detail-cx84121.html` | byte-for-byte copies of the plan captures |
| create | `pipeline/tests/fixtures/README.md` | capture date, URLs, HTTP status, raw sizes, facts, redactions, frozen rule |
| create | `pipeline/tests/test_fixtures_provenance.py` | the three cases below |

## Test first (red)

1. `test_fixtures_provenance.py::test_all_five_fixtures_are_present_and_utf8` — parametrised over the five names: each exists, is non-empty, its bytes start with `<!DOCTYPE`, and it decodes as UTF-8 without errors. Never assert byte sizes: the detail page is 222 117 bytes but 220 320 characters because Thai text is multi-byte.
2. `test_fixtures_provenance.py::test_provenance_readme_names_every_capture` — the copied README holds the date `2026-11-01`, all five names with a source URL or query per file, the raw sizes 169 109 / 169 349 / 174 631 / 105 / 222 117, and the pinned facts: pagers `1 Of 2`, `2 Of 2`, `71 Of 546`; states 4/5/7 and 0/1/15 and 12+4; `CX-84121` at ฿45,000, 65 sqm, 1 bed, 1 bath, floor 27, `Sa Thon` / `Bangkok`, `29/10/2569`; `13.7187616` / `100.5202356`; project `p00348`; the frozen rule.
3. `test_fixtures_provenance.py::test_fixtures_carry_the_redacted_detail_page` — no fixture contains `AIza`; the detail copy contains `REDACTED_GOOGLE_MAPS_KEY` twice, `REDACTED_CSRF_TOKEN` twice and `href="tel:REDACTED"` three times, and it has no live `href="tel:0` link (the search pages' site-wide contact widget is out of scope and is not asserted).

Expected red: `FileNotFoundError` for each parametrised case and for the missing README (7 failures).
Run: `. .venv/bin/activate && pytest pipeline/tests/test_fixtures_provenance.py -q`.

## Implement (green)

Copy the five files with `cp`, byte for byte — LF endings, no BOM, no edits — and never curl the live site. All five
returned HTTP 200.

The README repeats the per-file table (file, URL, HTTP, raw size, what it pins) and exactly the facts case 2 asserts.
It also records the applied redactions (Maps key ×2, CSRF token ×2, phone links ×3 → sentinels and `href="tel:REDACTED"`)
and the frozen rule: never re-capture or edit a fixture to make a test pass — if the site changed, stop and report.

Constraints: no test or command touches the network; the fixtures are frozen once committed; only the listed files are
committed, and the search pages' contact widget stays out of parse scope and out of documentation.

## Refactor / keep it DRY

One fixture directory, one provenance table; the test names the five files through one module-level tuple.

## Verification (how to test it)

```text
. .venv/bin/activate
pytest pipeline/tests/test_fixtures_provenance.py -q
pytest pipeline -q
```

Expected: `7 passed` for the file, `11 passed` for the suite (4 smoke + 7), six files in the directory, and a clean
`git status --short` for everything but the new paths.

## Out of scope (do not do)

- HTML parsing, `BeautifulSoup`, `UnitCard`, card discovery — 01-05 onward.
- `pipeline/tests/conftest.py` and loaders — 00-05; the detail parser — 04-01; synthetic fixtures, which stage 01
  creates as edited strings inside tests instead of files.
- Re-capturing, reformatting or trimming a fixture; changes under `plan/assets/`; images or `data/` files.

## Definition of done

- [ ] The three cases exist, named as written, and pass; red quoted in the commit body.
- [ ] `pytest pipeline -q` is 11 passed; `ruff check pipeline` clean; only the listed files are added.
- [ ] A re-run changes nothing; `status: done`; `## Result` filled; exactly one commit on the trunk as below.

## Commit

```text
00-04: copy the frozen html fixtures with provenance

- points: 0.5
- red: test_all_five_fixtures_are_present_and_utf8 failed (FileNotFoundError: pipeline/tests/fixtures/search-dev-p1.html)
- copy the five 2026-11-01 captures byte-for-byte and add the provenance README with facts and redactions
- suite: 11 passed
```

Files in the commit: the seven files above and this task file. One task = one commit, alone on the trunk (`main`), no
push, no branch.

## Traps and review notes

- Never assert a fixture's byte size or `len(text)` (the README's capture-time numbers are documentation, not a length
  assertion): page 1 is 169 109 B but 167 680 characters, the detail page 222 117 B but 220 320.
- Do not edit or reformat a fixture; `cp` only, and a re-run must leave `git status` unchanged.
- The error stub redirects via `top.location.href` to `/error/th`; `window.location.replace` belongs to the POST stub in
  `plan/01-discovery.md` §1. Assert `/error/th` and no cards, never the JS form.
- Never copy the contact widget's phone number into the README or tests, and never add a `*.html` ignore rule.
- Reviewer: `git ls-files pipeline/tests/fixtures` shows six files and `grep -rl 'AIza'` prints nothing.

## Plan samples are untrusted

The plan's sizes drift and the GET error stub uses `top.location.href`, not the `window.location.replace` quoted for the
POST stub: apply `CLAUDE.md` §6 and verify markers with `grep` before asserting.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
