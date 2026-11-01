---
id: 03-02
stage: 03
title: Name normalisation (NFKC, case, punctuation)
points: 0.5
owner: pipeline
status: todo
depends_on: [01-02]
files:
  - pipeline/src/connex_watch/normalize.py
  - pipeline/tests/test_normalize.py
---

# 03-02 · Name normalisation (NFKC, case, punctuation)

**Points:** 0.5 · **Stage:** 03 — Dataset builder · **Owner lane:** pipeline · **Depends on:** 01-02 · **Unblocks:** 03-03, 03-04, 03-05, 03-08

## Goal

Add `normalize_name(raw)` — the single comparison key for building names — so `A Tower` and `a tower` group as one building without fuzzy matching.

## Why this task exists

The site spells the same building differently from card to card (case, extra spaces, a trailing dot). The dataset must show one building per real building, and `plan/02-architecture.md` §6 rule P7 says the grouping key is the **normalised** name, while the name displayed is the most frequent original (03-03). Getting this key wrong either splits a building into two map bubbles or merges two buildings into one — both are visible product bugs. The key is also the input to `slugify`, so it must be stable across runs and across the NFC/NFD forms of Thai text.

## Current state (preconditions)

- 01-02 is done: `models.py` defines `UnitCard` / `PageResult`; the suite is green.
- `pipeline/src/connex_watch/normalize.py` does **not** exist; `pipeline/tests/test_normalize.py` does **not** exist.
- Nothing else in the package compares or rewrites building names yet.

## Read before you start

- `CLAUDE.md` §3 (TDD), §5 (DRY/YAGNI — no fuzzy matching, ever), §6 (samples untrusted).
- `plan/stage-03-dataset-builder.md` Task 3.1 items 1–3 — the three normalisation cases and their exact values.
- `plan/02-architecture.md` §3.1 (Building identity) and §6 P7 — the key is normalised text, not a FuzzyWuzzy-style score.
- `plan/01-discovery.md` §2 quirk 2 — names may contain commas, so internal punctuation must survive.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `pipeline/src/connex_watch/normalize.py` | `normalize_name` and its punctuation constant only |
| create | `pipeline/tests/test_normalize.py` | the five cases below |

Nothing else may change: no `slugify`, no `display_name`, no `parse_location` (03-03/03-04), no parser edits.

## Test first (red)

1. `test_normalize_name_collapses_whitespace_and_case` — table: `"  The  Bangkok   Sathorn "` → `the bangkok sathorn`; `"THE BANGKOK SATHORN"` → `the bangkok sathorn`; `""` → `""`; `"   \t\n"` → `""`.
2. `test_normalize_name_strips_end_punctuation` — `"The Bangkok Sathorn."` → `the bangkok sathorn`; `"The Bangkok Sathorn..."` → `the bangkok sathorn`; `",B Place;"` → `b place`; `"A Tower :"` → `a tower`.
3. `test_normalize_name_keeps_internal_punctuation` — `"Condo, A, B"` → `condo, a, b`; `"A-Tower"` → `a-tower` (the comma is what keeps CX-5 distinct from `Condo A B`).
4. `test_normalize_name_is_thai_stable_across_nfc_and_nfd` — the NFC and the NFD spelling of `เดอะบางกอกสาทร` both return the NFC string `เดอะบางกอกสาทร`; two calls are idempotent.
5. `test_normalize_name_casefolds_instead_of_lowercasing` — `"Straße"` → `strasse` (casefold, not `lower()`, which would keep `ß`).

Expected red: `ModuleNotFoundError: No module named 'connex_watch.normalize'`, then `ImportError: cannot import name 'normalize_name'` once the module exists empty. A syntax error is a broken test, not red.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_normalize.py -q`

## Implement (green)

`normalize_name(raw: str) -> str` in `normalize.py`, one sentence of docstring: NFKC-normalise, collapse every whitespace run to a single space, trim, casefold, then strip end punctuation repeatedly until nothing changes; returns `""` for a blank input. End punctuation is one module-level constant (`.,;:` — the characters seen on cards and lines), not literals scattered in the function. Internal punctuation and Thai combining marks are preserved. The function is pure: no logging, no I/O, no global state.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_normalize.py -q` — five green.

## Refactor / keep it DRY

The step order lives in exactly one function; the punctuation set in one constant. Later tasks (`slugify`, `display_name`, `parse_location`) import from this module — none of them may re-implement whitespace collapsing. Re-run the suite after any refactor.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_normalize.py -q
pytest pipeline -q
ruff check pipeline
```

Manual check: `python -c` is not needed — the five table cases carry the whole contract, and `ruff` proves no unused import crept into `normalize.py`.

## Out of scope (do not do)

- No `slugify`, `display_name` or `parse_location` — 03-03 and 03-04 add them to this same file.
- No fuzzy similarity, no `difflib`, no name aliasing, no transliteration.
- No change to `search_parser.py`: normalisation happens in the builder, never in the parser (stage 01 is frozen).
- No logging and no warnings — an empty key is a caller's problem (03-09 falls back).
- No new dependency.

## Definition of done

- [ ] All five cases exist, named as written, and pass; red run quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only the two new files.
- [ ] `normalize_name` has a one-sentence docstring stating its contract.
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
03-02: add building name normalisation key

- points: 0.5
- red: test_normalize_name_collapses_whitespace_and_case failed (ModuleNotFoundError)
- add normalize_name: NFKC, whitespace collapse, casefold, end-punctuation strip
- keep internal punctuation and Thai text intact
- suite: N passed
```

Files in the commit: `normalize.py`, `test_normalize.py` and this task file (status + Result). Nothing else.

## Traps and review notes

- `lower()` is not `casefold()`: assert `Straße` → `strasse` so the weaker call is caught.
- Stripping every comma would turn `Condo, A, B` into `condo a b` and could merge it with a different building; only **end** punctuation goes.
- NFKC is required before comparison: the NFD spelling of the same Thai name must not create a second building.
- Do not normalise the *display* name here — the most frequent original wins in 03-03, otherwise the UI shows a lowercased name.
- Reviewers: grep the repo for `.lower()` on building names and for `difflib` / `fuzz`; both must be absent.

## Plan samples are untrusted

`plan/stage-03-dataset-builder.md` shows the four helper signatures as a sketch. Verify the signatures against the repo before use and record any deviation under `## Decisions`; do not copy the sketch's assumptions about a `FuzzyMatcher` or about preserving case.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
