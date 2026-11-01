---
id: 00-03
stage: 00
title: Python package, pytest and ruff configuration
points: 1
owner: pipeline
status: todo
depends_on: [00-01]
files:
  - pipeline/pyproject.toml
  - pipeline/src/connex_watch/__init__.py
  - pipeline/tests/test_smoke.py
---

# 00-03 · Python package, pytest and ruff configuration

**Points:** 1 · **Stage:** 00 — Bootstrap · **Owner lane:** pipeline · **Depends on:** 00-01 · **Unblocks:** 00-04, 00-06, 01-01

## Goal

Make the pipeline installable and testable: one `pyproject.toml`, one importable `connex_watch`, one smoke test.

## Why this task exists

Every pipeline task from 01-01 on runs `pytest pipeline -q` and `ruff check pipeline` and imports `connex_watch`; without
a src-layout project, pins, a marker registry and lint configuration the first parser task invents all of it. The smoke
test pins two facts: the package exposes `0.1.0`, and importing it pulls in no HTTP client.

## Current state (preconditions)

- 00-01 is done and `git status` is clean. `pipeline/`, `.venv` and `pyproject.toml` do not exist; Python 3.12 is installed.

## Read before you start

- `CLAUDE.md` §3, §5, §6, §11 — test first, pinned dependencies, untrusted samples, venv at the root.
- `plan/stage-00-bootstrap.md` Task 0.2 — files, dependencies, markers, ruff rules; `plan/02-architecture.md` §4 (stack).

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `pipeline/pyproject.toml` | build metadata, pins, dev extras, pytest and ruff config |
| create | `pipeline/src/connex_watch/__init__.py` | docstring and `__version__` only |
| create | `pipeline/tests/test_smoke.py` | the four cases below |

## Test first (red)

1. `test_smoke.py::test_version_is_0_1_0` — `connex_watch.__version__` equals `"0.1.0"`.
2. `test_smoke.py::test_distribution_metadata_matches_package_version` — `importlib.metadata.version("connex-watch")` is `"0.1.0"` and equals `connex_watch.__version__`; the distribution is hyphenated, the import name underscored, so this catches the second copy drifting.
3. `test_smoke.py::test_console_script_is_declared_without_importing_it` — the `console_scripts` entry point `connex-watch` maps to `connex_watch.cli:main`; read metadata only, never import `connex_watch.cli` (02-13 creates it).
4. `test_smoke.py::test_importing_the_package_does_not_import_httpx` — a subprocess of `sys.executable` imports `connex_watch` and exits `1` if `"httpx"` is in `sys.modules`; the case asserts exit code `0`. No network.

Expected red: all four error during collection with `ModuleNotFoundError: No module named 'connex_watch'`. Scaffolding
exception (`CLAUDE.md` §3, the runner is the tool under test): create `.venv` at the repo root, install `pytest`,
`pytest-cov`, `ruff`, write the four cases, then create the project and reinstall editable. Run
`. .venv/bin/activate && pytest pipeline/tests/test_smoke.py -q`.

## Implement (green)

`pipeline/pyproject.toml`: setuptools backend, src layout (`packages.find` under `pipeline/src` only), name
`connex-watch`, version `0.1.0`, `requires-python = ">=3.12"`, no `readme`/`license` file reference.

Dependencies `httpx`, `beautifulsoup4`, `lxml` pinned `==` to what pip resolves, `pydantic` v2 (`>=2`), dev extra `dev`
with `pytest`, `pytest-cov`, `ruff` pinned the same way, and `[project.scripts]` declaring
`connex-watch = "connex_watch.cli:main"` (no `cli.py` until 02-13).

pytest: `testpaths = ["tests"]`, `addopts = "-q"`, marker `network: hits the live site`; ruff: `line-length = 100`,
`target-version = "py312"`, rules `E, F, I, UP, B` under `[tool.ruff.lint]`. `__init__.py`: docstring and
`__version__ = "0.1.0"`, no imports. Venv from the repo root: `python3 -m venv .venv`, `. .venv/bin/activate`,
`pip install -e "pipeline[dev]"`; never install inside `pipeline/` — `pyproject.toml` is the single record of the pins.

## Refactor / keep it DRY

One metadata source: `pyproject.toml` owns the version and tools, `__init__.py` the runtime string, case 2 syncs them.

## Verification (how to test it)

```text
. .venv/bin/activate
pytest pipeline/tests/test_smoke.py -q
pytest pipeline -q
ruff check pipeline
python -c "import sys, connex_watch; print(connex_watch.__version__, 'httpx' in sys.modules)"
```

Expected: `4 passed` for the file and the suite; `ruff` exits `0`; the import line prints `0.1.0 False`; `git status
--short` lists only the three files above.

## Out of scope (do not do)

- `models.py`, `urls.py`, parsers, fetchers, `cli.py`, `presets.toml` — 01-01 onward and 02-13; no stub CLI.
- `--cov` in `addopts`, a coverage threshold, mypy, tox/nox, CI, a lockfile tool, extra ruff rules or `ruff format`.
- A network test (02-16); `README.md` (00-02), `Makefile` (00-06), `web/` (05-01).

## Definition of done

- [ ] All four cases exist and pass, red quoted; `pytest pipeline -q` and `ruff check pipeline` are clean and the
      import check prints `0.1.0 False`.
- [ ] No network, fixtures frozen, only the three listed files committed; one commit as below; `status: done`.

## Commit

```text
00-03: add the python package and test configuration

- points: 1
- red: test_version_is_0_1_0 failed (ModuleNotFoundError: No module named 'connex_watch')
- add src-layout pyproject with pins, dev extras, ruff/pytest config; declare the script and __version__
- suite: 4 passed
```

Files in the commit: exactly the three files above and this task file (status and `## Result`). Committed alone on the
trunk (`main`): one task = one commit; do not push or branch.

## Traps and review notes

- Run `pytest pipeline` from the repo root, never inside `pipeline/`; keep `testpaths = ["tests"]`. No `readme` or
  `license` file reference: both point outside `pipeline/` and fail the build.
- Ruff's `select` sits under `[tool.ruff.lint]`; check the installed `ruff --version`, never paste the plan's snippet.
- Do not import httpx/bs4/pydantic in `__init__.py` (case 4 fails); `connex-watch` failing until 02-13 is expected.
  Do not invent pins: pin the versions pip installed, and keep them only in `pyproject.toml`.
- Reviewer: `pip show`, the ruff keys and `markers` via `grep`, and a fresh `python -c "import connex_watch"`.

## Plan samples are untrusted

The plan lists names but no versions and its ruff layout is stale: apply `CLAUDE.md` §6 and verify with `pip show`/`ruff --version`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
