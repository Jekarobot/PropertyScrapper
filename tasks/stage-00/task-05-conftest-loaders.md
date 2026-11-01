---
id: 00-05
stage: 00
title: Pytest fixture loaders
points: 0.5
owner: pipeline
status: todo
depends_on: [00-04]
files:
  - pipeline/tests/conftest.py
  - pipeline/tests/test_fixture_loaders.py
---

# 00-05 · Pytest fixture loaders

**Points:** 0.5 · **Stage:** 00 — Bootstrap · **Owner lane:** pipeline · **Depends on:** 00-04 · **Unblocks:** 00-06, 01-05

## Goal

Expose the five frozen fixtures to every pipeline test through one UTF-8 reader and five named pytest fixtures.

## Why this task exists

Parser tests from 01-05 on need the same five strings. Without a loader each test resolves its own paths and re-implements
reading, and one forgotten encoding breaks on a host whose locale is not UTF-8. `conftest.py` is the single seam: tests ask
for `search_dev_p1`, never for a path. The loader cases also pin that the copies are complete and that the error stub is
the small redirect page rather than a normal result page.

## Current state (preconditions)

- 00-04 is done: `pipeline/tests/fixtures/` holds the five HTML files and the provenance README; `pytest pipeline -q` is
  11 passed.
- `pipeline/tests/conftest.py` and `pipeline/tests/test_fixture_loaders.py` do not exist.

## Read before you start

- `plan/stage-00-bootstrap.md` Task 0.3 — the five fixture names and "no parsing logic in conftest".
- `plan/assets/fixtures/README.md` — what each copy pins; `plan/01-discovery.md` §3 — the end-of-data stub.
- `CLAUDE.md` §3 (fixtures frozen, no network) and §11 (`pytest pipeline` from the repo root).

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `pipeline/tests/conftest.py` | one UTF-8 reader plus the five fixtures |
| create | `pipeline/tests/test_fixture_loaders.py` | the three cases below |

## Test first (red)

1. `test_fixture_loaders.py::test_each_fixture_loads_as_utf8_text` — parametrised over the five fixtures and their marker: `search_dev_p1` → `CX-84121`; `search_dev_p2` → `2 Of 2`; `search_boundary_p71` → `71 Of 546`; `search_error_page` → `/error/th`; `detail_cx84121` → `center: {lat: 13.7187616, lng: 100.5202356}`. Each value is a `str`, not bytes, and contains its marker.
2. `test_fixture_loaders.py::test_search_pages_have_sixteen_card_markers` — parametrised over the three search fixtures: each contains exactly `16` occurrences of `class="condo-card"`, the card count pinned by the provenance README for pages 1, 2 and 71.
3. `test_fixture_loaders.py::test_error_page_fixture_is_the_small_redirect_stub` — `len(search_error_page) < 2048`, it contains `/error/th`, and it contains no `condo-card`.

Expected red: pytest reports `fixture 'search_dev_p1' not found` (and the other four names) before any assertion runs.
Run: `. .venv/bin/activate && pytest pipeline/tests/test_fixture_loaders.py -q`.

## Implement (green)

`pipeline/tests/conftest.py` defines one private reader: it resolves `Path(__file__).parent / "fixtures" / name` — never
the current working directory — and returns `read_text(encoding="utf-8")`, so a missing file raises `FileNotFoundError`
naming the path. Five session-scoped pytest fixtures are one-line wrappers over it: `search_dev_p1` for
`search-dev-p1.html`, `search_dev_p2`, `search_boundary_p71`, `search_error_page` for `search-error-out-of-range.html`,
and `detail_cx84121` for `detail-cx84121.html`. Each carries a one-sentence docstring naming its file.

No parsing, no BeautifulSoup, no assertions inside a fixture, no network, and no path constants exported for product code.

Constraints: no test or command touches the network; the fixtures are frozen and must not be edited or re-captured; only
the two files above are committed.

## Refactor / keep it DRY

The file read exists in exactly one function; the five fixtures are wrappers, not five copies of the read. The test
refers to fixtures by name, never to paths, and there is no second fixture directory.

## Verification (how to test it)

```text
. .venv/bin/activate
pytest pipeline/tests/test_fixture_loaders.py -q
pytest pipeline -q
grep -n 'encoding' pipeline/tests/conftest.py
```

Expected: `9 passed` for the file (5 + 3 + 1), `20 passed` for the suite (11 + 9), and the grep shows the reader passing
`encoding="utf-8"` explicitly.

## Out of scope (do not do)

- Parsing, `UnitCard`, `PageResult`, selectors and counting logic — 01-02 and 01-05 onward; the detail parser — 04-01.
- Fixture edits or re-capture; synthetic strings (stage 01 creates them inside tests); any network access.
- A repo-root `conftest.py`, `tests/__init__.py`, autouse fixtures, or exported fixture paths for product code.

## Definition of done

- [ ] The three cases exist, named as written, and pass; red quoted in the commit body.
- [ ] `pytest pipeline -q` is 20 passed; `ruff check pipeline` clean; only the two files are added.
- [ ] `status: done`; `## Result` filled; exactly one commit on the trunk with the message below.

## Commit

```text
00-05: add pytest loaders for the frozen html fixtures

- points: 0.5
- red: pytest failed with "fixture 'search_dev_p1' not found"
- add one utf-8 reader and five session fixtures in pipeline/tests/conftest.py
- add loader tests for markers, 16-card counts and the error stub
- suite: 20 passed
```

Files in the commit: the two files above and this task file. One task = one commit, alone on the trunk (`main`), no push,
no branch.

## Traps and review notes

- Pass `encoding="utf-8"` explicitly: `Path.read_text()` without it uses the locale and breaks on a non-UTF-8 host.
- Resolve paths from `__file__`, not the cwd: `pytest pipeline` runs from the repo root, but a test may be run elsewhere.
- Never assert sizes or `len(text)` beyond the stub's `< 2048` bound; the detail page's character count differs from its
  byte size because Thai text is multi-byte.
- The stub starts with a doctype and redirects via `top.location.href` to `/error/th`: assert that and no cards, not
  `window.location.replace`, which belongs to the POST stub in `plan/01-discovery.md` §1.
- No BeautifulSoup and no parsing: `str.count` is the only inspection these cases need.
- Reviewer: `grep -n 'httpx\|requests\|connex.in.th' pipeline/tests/conftest.py` prints nothing.

## Plan samples are untrusted

Task 0.3 lists the five names but no scope, no error behaviour and no marker: apply `CLAUDE.md` §6, verify each marker
with `grep` against the copies, and never move parser logic into `conftest.py`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
