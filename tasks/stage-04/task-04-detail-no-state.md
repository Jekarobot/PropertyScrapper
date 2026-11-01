---
id: 04-04
stage: 04
title: Detail page never yields an availability state
points: 0.5
owner: pipeline
status: todo
depends_on: [04-01]
files:
  - pipeline/src/connex_watch/detail_parser.py
  - pipeline/tests/test_detail_parser.py
---

# 04-04 · Detail page never yields an availability state

**Points:** 0.5 · **Stage:** 04 — Enrichment · **Owner lane:** pipeline · **Depends on:** 04-01 · **Unblocks:** nothing yet

## Goal

Make "the detail page carries no availability" an enforced boundary: `DetailInfo` has exactly six fields, rejects unknown ones, and exposes no state.

## Why this task exists

The project's central correctness rule is that a unit is `available` only when the search card says so. The detail page is a trap: its status line literally reads `Status : Available`, and the word also appears in navigation and buttons, yet a genuinely unavailable listing still shows it. Anyone who later adds a `state` field "because the data is right there" would silently corrupt the dataset. This task pins the negative: the detail parser produces coordinates and identity, nothing else, and the model refuses extra fields so the mistake fails loudly at parse time rather than at review time.

## Current state (preconditions)

- 04-01, 04-02, 04-03 done: `DetailInfo` (six fields), `parse_detail_page`, coordinate and identity rules, reasons and truncation tolerance are green (nineteen cases in `test_detail_parser.py`).
- The fixture still contains the visible status block in its `<h1>`-adjacent unit header, and the word `Available` elsewhere in navigation.
- `pytest pipeline -q` is green.

## Read before you start

- `CLAUDE.md` §3, §6 and §10 (state is never inferred from a detail page).
- `plan/01-discovery.md` §5 and §7 — "Do not infer state from the detail page", and the note that the word appears many times.
- `plan/02-architecture.md` §3.1 (tri-state enum) and §6 rule P5 (only `available` cards are published).
- `plan/stage-04-enrichment.md` Task 4.1 case 5.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/detail_parser.py` | `extra="forbid"` plus the boundary comment |
| modify | `pipeline/tests/test_detail_parser.py` | the four guard cases |

Nothing else may change: no change to fields, coordinates, identity, reasons, cache or CLI.

## Test first (red)

1. `test_detail_parser.py::test_fixture_states_available_but_parsed_model_has_no_state` — assert the fixture text contains the word `Available`, then assert the parsed model has no attribute `state`, no key containing `state`, `status` or `avail` in its dump, and no member of `DetailInfo.model_fields` named that way.
2. `...::test_detail_info_fields_are_exactly_the_six_contract_fields` — `list(DetailInfo.model_fields)` is exactly `["lat", "lng", "project_code", "building_name", "unit_code", "reason"]`, in that order.
3. `...::test_detail_info_forbids_unknown_fields` — constructing `DetailInfo(state="available")` raises pydantic's `ValidationError`; the same for `status="Available"`. This is the case that must be red: pydantic v2 ignores unknown keyword arguments by default.
4. `...::test_flipping_the_status_text_does_not_change_the_parse` — rewrite the page's `Status : Available` to `Status : Not available` and lowercase the nav occurrences → `model_dump()` is identical to the untouched fixture's dump.

Expected red: case 3 fails with "DID NOT RAISE ValidationError" (an extra field is silently accepted today); cases 1, 2 and 4 are green guards that must stay green.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_detail_parser.py -q`

## Implement (green)

- Add `extra="forbid"` to `DetailInfo`'s pydantic config (`ConfigDict`) and a one-line comment above the model: availability is never parsed here; state comes from the search card (rule P5).
- Change nothing else: no new field, no validator, no property named `state`, no helper that looks for status text.
- Keep the six-field order and defaults; the existing nineteen cases must stay green.
- Docstring addition: one sentence stating that the model is closed and deliberately has no availability field.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_detail_parser.py -q` — twenty-three green.

## Refactor / keep it DRY

There is nothing to extract: the guard is one config flag. If you feel the need for a `has_state` helper or a status regex, stop — that is the bug this task exists to prevent. Re-run the file after the change.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_detail_parser.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 23 passed, whole suite green, `ruff` exits 0. Manual check: `grep -in "state\|status" pipeline/src/connex_watch/detail_parser.py` shows only the boundary comment, never a parse.

## Out of scope (do not do)

- Deriving state, filtering units by state, or cross-checking state against the search card — stage 01/03 own state.
- Additive fields such as `description`, `images`, `agent`, `phone`, or a raw status string.
- Enforcing the same `extra="forbid"` on other models; only `DetailInfo` is in scope.
- Editing the fixture to remove the word `Available` — the word must stay, that is the point.

## Definition of done

- [ ] All four cases exist, named as written, and pass; the nineteen earlier cases still pass.
- [ ] Red run observed (case 3) and one failure line quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] Only the two files above plus this task file are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
04-04: forbid availability fields on DetailInfo

- points: 0.5
- red: test_detail_info_forbids_unknown_fields failed (DID NOT RAISE ValidationError)
- set extra="forbid" on DetailInfo and document the boundary
- guard that the fixture's Available text never becomes a field
- suite: N passed
```

Files in the commit: the two files above and this task file (status + Result). Nothing else.

## Traps and review notes

- A guard test with no production change is vacuous; the `extra="forbid"` config is what makes case 3 red now and green after. Do not delete case 3 because "it already passes".
- Do not assert on a word count in the page — the fixture may legitimately change that number, and the point is the model, not the HTML.
- Reviewers: `grep -n "extra"` shows one config line; `grep -in "available"` in the parser shows no match.

## Plan samples are untrusted

The plan states the rule in prose ("assert the parser never sets a state") and shows the model field list. Verify the pydantic v2 config API against the installed version (`CLAUDE.md` §6) before writing it; record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
