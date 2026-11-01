---
id: 01-07
stage: 01
title: Missing state block raises a located error
points: 0.5
owner: pipeline
status: todo
depends_on: [01-06]
files:
  - pipeline/src/connex_watch/search_parser.py
  - pipeline/tests/test_search_parser_states.py
---

# 01-07 · Missing state block raises a located error

**Points:** 0.5 · **Stage:** 01 — Search-page parser · **Owner lane:** pipeline · **Depends on:** 01-06 · **Unblocks:** none

## Goal

When a card carries no readable availability, fail with a `ParseError` that names the field and the card's position instead of crashing or inventing a state.

## Why this task exists

The state block is present on all 96 sampled cards, which is exactly why its absence must be loud: the likely cause is a class rename, and the failure mode we cannot accept is a card that silently becomes "not available". A located error also keeps the fix cheap — "card 7 has no state" tells the next engineer which markup to open, while a bare `AttributeError` from inside a selector tells nothing. The same treatment applies to a label the mapping does not recognise: that is a state error at a known position, not a stray `ValueError` escaping from the enum.

## Current state (preconditions)

- 01-06 is done: `parse_state(card_node)` reads the state label and `state_histogram` counts states; `test_search_parser_states.py` is green.
- `parse_state` assumes the state element exists, so a card without it fails with `AttributeError` on the missing element.
- `ParseError` already accepts a component name and an optional position (01-05); `pytest pipeline -q` is green.

## Read before you start

- `CLAUDE.md` §3 (red must fail for the stated reason), §9 (decisions go in the task file).
- `plan/01-discovery.md` §2 — the state rows and "exactly one per card" in the sample; §10 risks R1 (class renames) and R7 (a silent available claim).
- `plan/stage-01-search-parser.md` Task 1.2 item 5 — the raise-versus-`None` choice, with raising recommended.
- `plan/02-architecture.md` §3.1 — `UnitCard.state` is one of three values; there is no null state.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/search_parser.py` | the two guards in `parse_state` |
| modify | `pipeline/tests/test_search_parser_states.py` | the three cases below |

Nothing else may change: no new error type, no logging branch, no nullable state.

## Test first (red)

1. `test_missing_status_block_raises_a_located_state_error` — delete the whole `status-listing` block of the first card in `search_dev_p1` (string edit inside the test), rebuild the handles, call `parse_state` on position 1 → `ParseError` with `component == "state"`, `position == 1`, and a message containing `state` and `1`.
2. `test_missing_status_block_reports_the_middle_card_position` — the same deletion applied to the sixth card → `position == 6`; a hard-coded 1 or a vague "a card" message fails here.
3. `test_unknown_state_label_is_a_located_state_error` — change the third card's label text from `Available` to `Pending` → `ParseError` (not a bare `ValueError`), `component == "state"`, `position == 3`, and the message still contains `Pending` so the operator sees the new label.

Expected red symptom: `AttributeError: 'NoneType' object has no attribute 'get_text'` (or the equivalent failure of the selector) for cases 1 and 2, and `ValueError` escaping from `UnitState.from_label` for case 3. A failure that is already a `ParseError` but lacks the position means only half the work is done.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_states.py -q` — three red, seven green.

## Implement (green)

Two guards in `parse_state`, both reporting through the existing `ParseError`:

- state element absent (the status container or the labelled element inside it) → `ParseError("state", position)`; the message contains the component, the 1-based position, and says the card has no availability block;
- the label does not map → catch the `ValueError` from `UnitState.from_label` and re-raise as `ParseError("state", position)` preserving the offending label. Never fall back to a default member and never swallow it.

`parse_state` keeps returning `UnitState` and never `None`. No new exception class, no `Optional` in the signature, no logging. Update the docstring to state both error conditions.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_states.py -q` — ten green.

## Refactor / keep it DRY

Both branches produce the same error shape from one small helper ("raise a state error at this position") rather than duplicated message construction. `UnitState.from_label` stays the only label table. Re-run the suite after the refactor.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_search_parser_states.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Manual check: trigger case 1 in a REPL and read the exception — it names `state` and `1` — then confirm the untouched fixture still parses all 16 states.

## Out of scope (do not do)

- Recovering a state from another element, the image `alt` text, or a detail page; missing means error.
- A nullable `state` on `UnitCard` (01-02 owns the model), a fourth enum member, or a default of `not_available`; page-level assembly — 01-17.
- Broad `except Exception` around the whole card parse, a warning-and-continue branch, or collecting bad cards into a side list.

## Definition of done

- [ ] All three cases exist, named as written, and pass; the red run is quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `search_parser.py`, `test_search_parser_states.py`, this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
01-07: raise a located error for a missing state

- points: 0.5
- red: test_missing_status_block_raises_a_located_state_error failed (AttributeError: NoneType)
- guard the state lookup and wrap unknown labels in ParseError("state", position)
- suite: 44 passed
```

Files in the commit: `search_parser.py`, `test_search_parser_states.py`, this task file. Nothing else.

## Traps and review notes

- Do not catch `ValueError` broadly: `ParseError` is a `ValueError`, so an outer `except ValueError` re-wraps your own errors and doubles the message.
- Take the position from the card handle (01-05), not from an enumeration inside the state function, or it drifts once a caller filters cards.
- Do not put `repr()` of a BeautifulSoup tag in the message; it embeds the whole card.
- Reviewers: grep `parse_state` for `return None` and `except Exception` — both must be absent.

## Plan samples are untrusted

`plan/stage-01-search-parser.md` suggests either raising `ParseError("state")` or returning a `None` state, recommending the raise. This task takes that route and adds the position, which the sample does not show. Verify the real exception text against the installed BeautifulSoup version and record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
