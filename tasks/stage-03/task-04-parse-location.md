---
id: 03-04
stage: 03
title: Location tuple from split parts
points: 0.5
owner: pipeline
status: todo
depends_on: [03-02]
files:
  - pipeline/src/connex_watch/normalize.py
  - pipeline/tests/test_normalize.py
---

# 03-04 · Location tuple from split parts

**Points:** 0.5 · **Stage:** 03 — Dataset builder · **Owner lane:** pipeline · **Depends on:** 03-02 · **Unblocks:** nothing in this stage

## Goal

Add `parse_location(parts)` — the one implementation of "split a location line from the right" — returning `(building, district, province)` with `None` where the site omitted a level.

## Why this task exists

The location line is `Building , District , Province` and building names contain commas (`Condo, A, B`), so a left-to-right split corrupts the name (`plan/01-discovery.md` §2 quirk 2). This task pins the right-split contract as a pure function over already-split, trimmed parts, with the exact fallbacks for the 2-part line that exists in the wild and the 1-part line that a layout change could produce. Later stages must never re-derive district or province with a second splitter; if the parser is ever refactored to share this helper, it imports this one.

## Current state (preconditions)

- 03-02 is done: `normalize.py` holds `normalize_name`; `test_normalize.py` has five passing cases.
- 01-13 already parses the location line inside `search_parser.py` and stores `district` / `province` on each card, so stage 03 reads those fields and this helper has no builder caller yet.

## Read before you start

- `CLAUDE.md` §3 (TDD), §5 (one implementation per rule), §6, §7.
- `plan/01-discovery.md` §2 quirk 2 — the exact split rule and the 2-part warning case; §2 location row of the selector table.
- `plan/stage-03-dataset-builder.md` Task 3.1 item 6 — the four expected tuples.
- `pipeline/src/connex_watch/search_parser.py` — how 01-13 already splits the line, so you can prove this function agrees.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/normalize.py` | add `parse_location` only |
| modify | `pipeline/tests/test_normalize.py` | the five cases below |

Nothing else may change: no `search_parser.py` edit, no builder call, no new module.

## Test first (red)

1. `test_parse_location_three_parts` — `["A, B Condo", "Khlong Toei", "Bangkok"]` → `("A, B Condo", "Khlong Toei", "Bangkok")`, with each part trimmed (`" A, B Condo "` gives the same result).
2. `test_parse_location_two_parts_has_no_district` — `["X", "Bangkok"]` → `("X", None, "Bangkok")`; `["A Tower", "", "Bangkok"]` → the same tuple, because empty parts are dropped before mapping.
3. `test_parse_location_one_part_has_no_district_or_province` — `["X"]` → `("X", None, None)`.
4. `test_parse_location_empty_returns_empty_building` — `[]` → `("", None, None)`; `["", "  "]` → `("", None, None)`.
5. `test_parse_location_joins_extra_parts_from_the_right` — `["Condo", "A", "B", "Khlong Toei", "Bangkok"]` → `("Condo , A , B", "Khlong Toei", "Bangkok")`, joined with the site's ` , ` separator.

Expected red: `ImportError: cannot import name 'parse_location'` from `connex_watch.normalize`. A tuple-shape failure means the parts were mapped left-to-right — that is the bug this task exists to prevent.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_normalize.py -q`

## Implement (green)

`parse_location(parts: list[str]) -> tuple[str, str | None, str | None]`: trim every part, drop the empty ones, then map from the right — last part is `province`, next is `district`, everything remaining is the building name joined with `" , "`. Fewer than three parts yields `None` for the missing levels; no parts at all yields `("", None, None)`. No exception for short input: a 2-part line is data, not an error. One sentence of docstring. Pure, no logging (the parser already warned at parse time).

Run: `. .venv/bin/activate && pytest pipeline/tests/test_normalize.py -q` — seventeen green.

## Refactor / keep it DRY

This is the only right-splitter in the repo's future: if `search_parser.py` ever needs it, it imports `parse_location` instead of growing a second copy — but that refactor is **not** part of this task (stage 01 is frozen, one task = one commit). No helper that returns a dict as well; the tuple is the contract.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_normalize.py -q
pytest pipeline -q
ruff check pipeline
```

Manual check: compare case 1's output with the `district`/`province` values that 01-13 stores for the DEV fixture's `Condo, A, B`-style line; they must agree.

## Out of scope (do not do)

- No edits to `search_parser.py`, no relocation of 01-13's logic, no parser test changes.
- No district/province voting or null handling per building — 03-10 owns the vote.
- No geography, no district whitelist, no province normalisation.
- No exception type: short input is not an error.
- No second splitter, no `str.split(",")` anywhere in the builder.

## Definition of done

- [ ] All five cases exist, named as written, and pass; red run quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `normalize.py` and `test_normalize.py`.
- [ ] `parse_location` has a one-sentence docstring with its input → output contract.
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
03-04: add parse_location right-split helper

- points: 0.5
- red: test_parse_location_three_parts failed (ImportError: cannot import name 'parse_location')
- add parse_location: trim, drop empty parts, map province/district/building from the right
- return None for missing levels instead of raising
- suite: N passed
```

Files in the commit: `normalize.py`, `test_normalize.py` and this task file. Nothing else.

## Traps and review notes

- Splitting left-to-right passes the 3-part case and fails only the 5-part case — keep case 5.
- Joining with a bare `","` produces `Condo,A,B`; the site's separator is `" , "` and the test pins it.
- Do not raise on 2 parts: `plan/01-discovery.md` §2 says such a line exists in the wild.
- Do not wire this into the builder here; there is no stage-03 caller while the raw layer stores `district`/`province` per card. If a reviewer asks for a caller, the answer is 03-10's vote.
- Reviewers: grep for `split(",")` and for a second province assignment outside this function and `search_parser.py`.

## Plan samples are untrusted

The plan's sketch returns `(" , ".join(parts[:-2]), parts[-2], parts[-1])` guarded only by a length check — it would raise or mis-join on a 2-part line. Follow this task and the five cases; record the deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
