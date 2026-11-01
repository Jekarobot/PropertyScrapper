---
id: 03-03
stage: 03
title: Slug and display-name selection
points: 1
owner: pipeline
status: todo
depends_on: [03-02]
files:
  - pipeline/src/connex_watch/normalize.py
  - pipeline/tests/test_normalize.py
---

# 03-03 · Slug and display-name selection

**Points:** 1 · **Stage:** 03 — Dataset builder · **Owner lane:** pipeline · **Depends on:** 03-02 · **Unblocks:** 03-08

## Goal

Add `slugify(name)` and `display_name(variants)` to `normalize.py`: a URL-safe, Thai-preserving identifier and the one original spelling a building is shown under.

## Why this task exists

Grouping produces a set of spellings for one building (`A Tower`, `a tower`) but the dataset needs two different things from them: a stable `id` other layers can key on, and a human string for the UI. The id must survive JSON, ordering and the map (`plan/02-architecture.md` §3.3), must never be transliterated (Thai names must stay Thai), and must be deterministic so two runs produce byte-identical files (P8). The display name must look like the site, not like the grouping key, which is why it is picked from the originals and not from `normalize_name`.

## Current state (preconditions)

- 03-02 is done: `normalize.py` exists with `normalize_name` and its end-punctuation constant; five normalisation tests pass.
- Nothing produces a slug or chooses a display name yet.

## Read before you start

- `CLAUDE.md` §3 (TDD), §5 (YAGNI — no transliteration, no slug library), §6.
- `plan/stage-03-dataset-builder.md` Task 3.1 items 4–5 — slug rules, Thai preservation, 80-char cap, display-name tie rule.
- `plan/02-architecture.md` §3.3 — `buildings[].id` and `buildings[].name`; §6 P8 — deterministic output.
- `plan/01-discovery.md` §2 quirk 2 and the fixture table — a real display name is `The Bangkok Sathorn`, not its key.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/normalize.py` | add `slugify` and `display_name`, no second normaliser |
| modify | `pipeline/tests/test_normalize.py` | the seven cases below |

Nothing else may change: no builder code, no `models.py`, no transliteration table.

## Test first (red)

1. `test_slugify_basic` — table: `"The Bangkok Sathorn"` → `the-bangkok-sathorn`; `"A, B Condo"` → `a-b-condo`; `"  A  Tower  "` → `a-tower`; no leading or trailing hyphen in any result.
2. `test_slugify_preserves_thai` — `"เดอะบางกอกสาทร"` → `เดอะบางกอกสาทร` unchanged (not transliterated); `"เดอะ บางกอก"` → `เดอะ-บางกอก`.
3. `test_slugify_truncates_to_eighty_chars` — a name built from 20 repetitions of `tower ` yields a slug of at most 80 characters, never ending in `-`, equal to the untruncated slug's first 80 characters with trailing hyphens removed.
4. `test_slugify_of_blank_is_empty` — `""` → `""`; `" , "` → `""`; `"--"` → `""` (03-09 defines the fallback id for this case).
5. `test_display_name_takes_the_most_frequent_original` — `["The Bangkok Sathorn", "the bangkok sathorn", "The Bangkok Sathorn"]` → `The Bangkok Sathorn`; `["a tower", "a tower", "A Tower"]` → `a tower`.
6. `test_display_name_tie_breaks_lexicographically` — `["alpha", "Alpha"]` → `Alpha`; `["b", "a", "B", "A"]` → `A` (code-point order: uppercase first).
7. `test_display_name_of_empty_iterable_is_empty` — `[]` → `""`, and each variant is trimmed of surrounding whitespace before counting.

Expected red: `ImportError: cannot import name 'slugify'` (and `display_name`) from `connex_watch.normalize`. If a case fails on a value, fix the test only after proving the expectation with Python's own behaviour.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_normalize.py -q`

## Implement (green)

`slugify(name: str) -> str`: NFKC, casefold, replace every run of characters that are neither a letter nor a digit (Unicode-aware, so Thai survives) with a single `-`, trim `-` at both ends, truncate to 80 characters and trim a trailing `-` again. No library, no `unicodedata.normalize` chain beyond NFKC, no transliteration, no collision handling (that is 03-09).

`display_name(variants: Iterable[str]) -> str`: count the variants as given (after trimming surrounding whitespace), return the one with the highest count; on a tie return the smallest by code point (`min`), which is stable and order-independent. Return `""` for an empty iterable.

One sentence of docstring each. Both are pure; neither logs (a warning about collisions belongs to 03-09).

Run: `. .venv/bin/activate && pytest pipeline/tests/test_normalize.py -q` — twelve green (five from 03-02 plus seven here).

## Refactor / keep it DRY

`slugify` must not re-implement whitespace collapsing or casefolding: build it on NFKC plus one regex pass, and if `normalize_name` already produced a trimmed key the caller may pass either form — document which. `display_name` counts a `Counter`, not nested loops. No second `-`-trim helper: one small private function used by both truncation paths.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_normalize.py -q
pytest pipeline -q
ruff check pipeline
```

Manual check: a Thai name slug printed by an ad-hoc test run contains Thai characters and no `%`-escapes.

## Out of scope (do not do)

- No collision suffix and no empty-slug fallback — 03-09 owns id assignment.
- No transliteration, no `python-slugify`, no `unidecode`, no abbreviation of `Tower`/`Condo`.
- No locale-dependent casing (`lower()` is not enough; casefold is the contract).
- No change to `normalize_name`'s behaviour — its five tests must stay green untouched.

## Definition of done

- [ ] All seven cases exist, named as written, and pass; red run quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `normalize.py` and `test_normalize.py`.
- [ ] Both functions have a one-sentence docstring; no new dependency in `pyproject.toml`.
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
03-03: add slug and display-name selection

- points: 1
- red: test_slugify_basic failed (ImportError: cannot import name 'slugify')
- add slugify: hyphen separator, Thai preserved, 80-char cap
- add display_name: most frequent original, lexicographic tie-break
- suite: N passed
```

Files in the commit: `normalize.py`, `test_normalize.py` and this task file. Nothing else.

## Traps and review notes

- Do not truncate by characters *then* leave a trailing `-`; the test asserts the cleaned 80-char result.
- Thai text must not be dropped by an ASCII-only `\w` regex — use Unicode classes and prove it with case 2.
- Counting normalised variants in `display_name` would always return an already-lowercased name; count the originals.
- `min()` on code points is deliberate: `Alpha` beats `alpha`. A reviewer checks the tie case, not the happy path.
- Reviewers: grep for `unidecode`, `slugify(` imports and `lower()`; all three must be absent.

## Plan samples are untrusted

The plan sketches `slugify` as a one-liner over `re.sub` with an ASCII class — that sketch drops Thai and ignores the 80-char cap. Follow this task and the tests; record the deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
