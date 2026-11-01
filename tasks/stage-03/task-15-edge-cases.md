---
id: 03-15
stage: 03
title: Grouping edge cases and degraded input
points: 1
owner: pipeline
status: todo
depends_on: [03-11]
files:
  - pipeline/src/connex_watch/build_dataset.py
  - pipeline/tests/test_build_edge.py
---

# 03-15 · Grouping edge cases and degraded input

**Points:** 1 · **Stage:** 03 — Dataset builder · **Owner lane:** pipeline · **Depends on:** 03-11 · **Unblocks:** nothing in this stage

## Goal

Pin the five boundary behaviours of the finished builder — a check-only building never appears, price bounds are not re-filtered, a trailing `Tower` never merges, unit urls are stored verbatim, and a 0-byte page degrades to `partial` instead of a crash.

## Why this task exists

Each of these is a plausible "helpful" addition that would quietly corrupt the product. Re-filtering prices by the search spec hides units the site already returned; fuzzy-merging `Sathorn` with `Sathorn Tower` puts two buildings in one bubble; rebuilding a url drops the query string the site used and makes the link 404; dropping a building whose units are all `check_availability` is correct but easy to implement as "drop units only", leaving an empty building row; and one truncated page file in a 3-page run must not abort the nightly build when 99 % of the data is usable. The plan asks for these as tests plus only the fixes they demand (`plan/stage-03-dataset-builder.md` Task 3.5).

## Current state (preconditions)

- 03-11 is done: `build_dataset` produces the run-mini dataset; `stats.partial` is derived from the manifest and `RawRun.skipped_pages`.
- 03-12…03-14 are done: `validate_dataset`, `check_dataset`, `build` and `check` exist.
- `pipeline/tests/test_build_edge.py` does not exist.

## Read before you start

- `CLAUDE.md` §3 (TDD), §5 (YAGNI: no fuzzy matching, no url rebuilding), §6.
- `plan/stage-03-dataset-builder.md` Task 3.5 — the five cases; Acceptance criteria (`run-mini` → 3 buildings).
- `plan/02-architecture.md` §3.3 (urls are stored, not built), §6 P5, P8, P9; §3.2 (a page file is written atomically, so a 0-byte file means a crashed writer).
- `plan/01-discovery.md` §5 — availability only exists on the search card, so a check-only building has no publishable unit.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/build_dataset.py` | the page-skip path in `load_run` and the `partial` derivation only |
| create | `pipeline/tests/test_build_edge.py` | the six cases below |

Nothing else may change: no validator, no CLI, no fixture edit (tests copy run-mini into `tmp_path` and mutate the copy).

## Test first (red)

1. `test_check_only_building_is_absent` — a copy of run-mini in which all of A Tower's available cards are flipped to `check_availability` yields no building named `A Tower`, keeps `B Place` and `A, B Condo`, and counts the flipped cards in `stats.excluded.check_availability`.
2. `test_price_bounds_are_not_refiltered` — a unit at exactly `25000` and one at exactly `50000` (the FULL spec's bounds) both survive, and so does a unit at `24000`: the builder applies **no** price filter, because the site already did.
3. `test_trailing_tower_is_not_merged` — `The Bangkok Sathorn` and `The Bangkok Sathorn Tower` produce two buildings with ids `the-bangkok-sathorn` and `the-bangkok-sathorn-tower`.
4. `test_unit_url_query_string_is_stored_verbatim` — a url `https://connex.in.th/en/detail/x-cx-10?ref=search&page=2` appears byte-for-byte in the JSON and still passes `validate_dataset` (the prefix rule).
5. `test_zero_byte_page_is_skipped_with_warning_and_sets_partial` — truncating `page-0002.json` to 0 bytes still builds: `pagesScanned == 2`, `skipped_pages == [2]`, `stats.partial is True`, four units survive, and one WARNING names the file.
6. `test_partial_follows_the_manifest_stop_reason` — a copy whose manifest has `stopReason aborted` (all three pages readable) gives `stats.partial is True`, while `last_page` with `errorCount 0` and no skipped page gives `False`.

Expected red: `FileNotFoundError` for `test_build_edge.py`'s own path constants, then `TypeError`/`KeyError` because `RawRun` has no `skipped_pages` handling and `partial` ignores the stop reason (cases 5 and 6).

Run: `. .venv/bin/activate && pytest pipeline/tests/test_build_edge.py -q`

## Implement (green)

In `load_run`: a page file whose size is 0 bytes is skipped, the page number is appended to `skipped_pages`, and one WARNING per file names it; unreadable JSON is still an error (that is a bug, not a crash artefact). `pagesScanned` counts files actually read. In `build_dataset`: `stats.partial` is true when `stopReason != "last_page"`, or `errorCount > 0`, or `skipped_pages` is non-empty. No other behaviour changes — cases 1–4 must pass on the existing implementation; if one does not, fix the builder, not the test.

The copied fixtures live under `tmp_path` and are mutated with the `json` module; the committed `run-mini` stays untouched.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_build_edge.py -q` — six green.

## Refactor / keep it DRY

The skip decision lives in one place (`load_run`) and the partial decision in one place (`build_dataset`); `stats.partial` must not be recomputed in the CLI or the validator. The zero-byte check uses the file size, not a `try/except JSONDecodeError`, so a corrupt-but-non-empty page still fails loudly (03-13 exit 4).

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_build_edge.py -q
pytest pipeline -q
connex-watch build --raw pipeline/tests/fixtures/run-mini --out /tmp/ds.json
connex-watch check --dataset /tmp/ds.json
ruff check pipeline
```

Manual check: the build command on the untouched fixture still prints `units=5 buildings=3 pages=3 partial=false` and `check` exits 0; a temporary 0-byte copy prints `partial=true`, and `check` on that file exits 1 by design (P9).

## Out of scope (do not do)

- No fuzzy matching, no prefix/suffix stripping, no `difflib`, no alias table.
- No url rebuilding, no query-string stripping, no `urllib.parse` normalisation of stored urls.
- No price/size re-filtering in the builder for any bound.
- No "repair" of a 0-byte page, no retry, no second read.
- No new dependency and no fixture edits (mutate copies in `tmp_path`).

## Definition of done

- [ ] All six cases exist, named as written, and pass; red run quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `build_dataset.py` and the new `test_build_edge.py`.
- [ ] `pipeline/tests/fixtures/run-mini/` is unchanged (`git diff --stat` on it is empty).
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
03-15: builder edge cases and degraded input

- points: 1
- red: test_zero_byte_page_is_skipped_with_warning_and_sets_partial failed (TypeError)
- skip 0-byte page files with a warning and count them as skipped
- derive stats.partial from stop reason, error count and skipped pages
- pin: no price refilter, no fuzzy merge, urls verbatim, check-only building absent
- suite: N passed
```

Files in the commit: `build_dataset.py`, `test_build_edge.py` and this task file. Nothing else.

## Traps and review notes

- Do not drop a building by filtering units late: a group with zero units must never be created, or the dataset grows a `unitCount 0` row.
- `24000` surviving is deliberate — adding a `minprice`/`maxprice` filter "for safety" is the mistake case 2 forbids.
- A 0-byte page and a corrupt page are different failures: skip the first, exit 4 on the second.
- `skipped_pages` must be part of `RawRun`, not a module-level global, or two builds in one test process interfere.
- Reviewers: grep for `len(page_bytes) == 0`/`st_size == 0` (one place), for `unquote`/`urlsplit` (none) and for `fuzz` (none).

## Plan samples are untrusted

The plan lists these cases without expected numbers and calls the last one "0-byte page files → skipped"; the `pagesScanned = 2` and four-surviving-units values above are derived from run-mini. Follow this task and record any deviation under `## Decisions`. Do not re-capture a fixture to make a test pass.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
