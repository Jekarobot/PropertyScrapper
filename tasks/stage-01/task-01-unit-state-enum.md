---
id: 01-01
stage: 01
title: UnitState enum and label mapping
points: 0.5
owner: pipeline
status: todo
depends_on: [00-03]
files:
  - pipeline/src/connex_watch/models.py
  - pipeline/tests/test_models.py
---

# 01-01 · UnitState enum and label mapping

**Points:** 0.5 · **Stage:** 01 — Search-page parser · **Owner lane:** pipeline · **Depends on:**
00-03 · **Unblocks:** 01-02, 01-06, 03-05

## Goal

Add the three-value availability enum and the single mapping from the site's English labels to it, so
every later layer has one vocabulary for availability.

## Why this task exists

The site marks each result card with one of three CSS classes, and the only human-readable version of
that state is a label inside the card (`Available`, `Not available`, `Check Availability`). The
product must never present "check availability" as "available" — that is the project's central
correctness rule (`plan/01-discovery.md` §5, risk R7). So the state must become a typed enum with an
explicit, testable conversion, before any HTML is parsed. Every parser, the dataset builder and the
web types depend on these three string values never changing.

## Current state (preconditions)

- Task 00-03 is done: `pipeline/pyproject.toml` exists, the package `connex_watch` imports, `pytest`
  and `ruff` are configured, `.venv` exists at the repo root.
- `pipeline/src/connex_watch/models.py` does **not** exist yet. `pipeline/tests/test_models.py` does
  **not** exist yet.
- `pytest pipeline -q` is green (one smoke test).

## Read before you start

- `CLAUDE.md` §2 (workflow), §3 (TDD rules), §4 (story points), §6 (plan samples are untrusted).
- `plan/02-architecture.md` §3.1 — the domain objects table and the availability enum paragraph.
- `plan/01-discovery.md` §2 (selector table, rows "state" and "state label") and §5 — why the tri-state
  exists and why it is not a boolean.
- `plan/90-test-matrix.md` row P-03 — the requirement this task satisfies.
- `plan/stage-01-search-parser.md` Task 1.1 — the original wording.
- The fixture `plan/assets/fixtures/search-dev-p1.html` — find the three state `<div>`s with their
  `<span>` labels and read the exact strings; do not assume capitalisation.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `pipeline/src/connex_watch/models.py` | the enum and its label mapping only |
| create | `pipeline/tests/test_models.py` | the four cases below |

Nothing else may change in this task (no `__init__.py` exports, no parsers, no pydantic models yet).

## Test first (red)

Write these tests before creating `models.py`. Use exact expected values; no fuzzy assertions.

1. `test_models.py::test_state_has_exactly_three_string_values` — the enum exposes exactly three
   members whose values are the strings `available`, `check_availability`, `not_available`, in that
   order; iterating the enum yields three items; each member is a `str` instance (so it survives JSON
   and pydantic serialisation as its value, not as a member name).
2. `test_models.py::test_from_label_maps_the_three_site_labels` — parameterised over the pairs
   (`Available` → `available`, `Not available` → `not_available`, `Check Availability` →
   `check_availability`); each call returns the corresponding member.
3. `test_models.py::test_from_label_strips_surrounding_whitespace` — `"  Available\n"` maps to
   `available` (the site's span text can carry whitespace).
4. `test_models.py::test_from_label_unknown_label_raises_with_label_in_message` — `"Pending"` raises
   `ValueError`; the message contains `Pending` and names the acceptable labels; `"available"`
   (lowercase) also raises, because labels are matched exactly after stripping and a changed site
   label must fail loudly rather than be guessed.

Expected red symptom: `ModuleNotFoundError: No module named 'connex_watch.models'` for case 1, and
`ImportError: cannot import name 'UnitState'` once the module exists but is empty. If the failure is a
syntax error or a typo, fix the test before continuing.

Run, from the repo root with the venv active:

`. .venv/bin/activate && pytest pipeline/tests/test_models.py -q`

Confirm all four cases are red for the stated reason and copy one failure line.

## Implement (green)

Create `pipeline/src/connex_watch/models.py` with one public item for now:

- `UnitState` — a string enum (`str`-valued) with exactly the three members above and the exact
  lowercase values. It must serialise to its value and compare equal to the plain string.
- `UnitState.from_label(label)` — classmethod/staticmethod that strips surrounding whitespace from the
  input, compares it exactly against the three canonical labels, and returns the member. The canonical
  labels are the ones printed on the cards: `Available`, `Not available`, `Check Availability`. Any
  other input, including a different letter case, raises `ValueError` whose message contains the
  offending label and lists the accepted labels.
- The label → member mapping exists **once**, in this module. No other module may hardcode label
  strings or the enum values; later tasks import `UnitState`.
- One-sentence docstring on `from_label` stating its contract (input → output → error).

Do not add an `UNKNOWN` member, a `from_class_name()` helper, or logging here. Do not import
BeautifulSoup, pydantic, or anything network-related in this task.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_models.py -q` — four green.

## Refactor / keep it DRY

The mapping table must be a single structure used by `from_label` (not three if-branches plus a dict).
No second copy of the values anywhere. Re-run the tests after any refactor.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_models.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 4 passed in the first command; the whole pipeline suite green; `ruff` exits 0.

## Out of scope (do not do)

- `UnitCard`, `PageResult`, `SearchSpec`, or any pydantic model — task 01-02 creates them.
- HTML parsing, selectors, BeautifulSoup — tasks 01-05 onward.
- Dataset-level filtering or statistics — stage 03.
- Exporting the enum from `connex_watch/__init__.py` or adding `__all__` elsewhere.
- Logging or warnings when a label is unknown: raising is the contract.

## Definition of done

- [ ] All four cases exist in `pipeline/tests/test_models.py`, named as written, and pass.
- [ ] Red run observed: `ModuleNotFoundError`/`ImportError` line quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `models.py` and `test_models.py`.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
01-01: add unit state enum and label mapping

- points: 0.5
- red: test_state_has_exactly_three_string_values failed (ModuleNotFoundError)
- add UnitState with three string values and exact label mapping
- unknown labels raise ValueError naming the label
- suite: 5 passed
```

Files in the commit: `pipeline/src/connex_watch/models.py`, `pipeline/tests/test_models.py`, and this
task file (status + Result). Nothing else.

## Traps and review notes

- Do not use `auto()` values: the stored strings must be exactly `available`, `check_availability`,
  `not_available`; they travel into `dataset.json` and the web types.
- Do not add a fourth member for unknown states. A silent `unknown` member becomes a silent
  "not available" downstream, which is the failure mode this project exists to avoid.
- Do not make the label match case-insensitive or startswith-based: a changed site label must produce
  a loud error, not a guess.
- Do not put the canonical labels in the test only — the mapping must live in `models.py`.
- Reviewers: grep for the label strings outside `models.py`; there must be none after this task.

## Plan samples are untrusted

The plan mentions enum members and labels but contains no code for this task. If you consult any
snippet in `plan/`, verify it against the installed Python/pydantic versions (see `CLAUDE.md` §6)
before using it. Record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
