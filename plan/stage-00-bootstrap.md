# Stage 00 — Bootstrap the repository

**Goal:** an empty-but-green repo: two projects (Python pipeline, Vite web), real fixtures, and one
command (`make check`) that proves the toolchain works. No product logic yet.

**Why first:** every later stage assumes `pytest`, `vitest`, `ruff`, `tsc` and the fixture loaders
exist. Doing this incrementally causes duplicated config and inconsistent test conventions.

**Depends on:** nothing. **Est:** 25 min. **Agent:** A (pipeline owner); web scaffold can be handed
to agent B, but it is small enough to keep here.

---

## Task 0.1 — Repo state, remotes, ignore rules

**Starting state (already true):** the repository exists and is connected —
`origin = git@github.com:Jekarobot/PropertyScrapper.git`, branch `main`, one commit containing
`plan/` only. GitHub auth is the owner's SSH key (`ssh -T git@github.com` → `Hi Jekarobot!`).

1. Verify, do not re-create: `git remote -v` → `origin` fetch+push; `git log --oneline` → 1 commit;
   `git status` → clean. **Never** `git init` again and **never** force-push `main`.
2. Work on a branch: `git switch -c stage-00`.
3. Create `README.md` skeleton (headings in `plan/stage-08-hardening-ops.md` task 8.5) and
   `data/.gitkeep`.
4. `.gitignore` must contain at minimum:

```text
.venv/
__pycache__/
*.pyc
.pytest_cache/
.ruff_cache/
node_modules/
web/dist/
web/playwright-report/
web/test-results/
web/public/data/
data/raw/
data/runs/
data/cache/
.probe/
logs/
```

5. Commit and push: `git push -u origin stage-00`. Merging to `main` is the owner's call (PR or
   `git switch main && git merge --no-ff stage-00`).

**Verify:** `git status` clean after commit; `git ls-remote origin` lists `refs/heads/stage-00`;
`git ls-files` on the branch shows `.gitignore`, `README.md`, `data/.gitkeep`, `plan/**`.

## Task 0.2 — Python project that can run one test

**Files:** `pipeline/pyproject.toml`, `pipeline/src/connex_watch/__init__.py`,
`pipeline/tests/test_smoke.py`.

1. `pyproject.toml`: `requires-python = ">=3.12"`, src layout, dependencies
   `httpx`, `beautifulsoup4`, `lxml`, `pydantic>=2`, dev extras `pytest`, `pytest-cov`, `ruff`.
   Add `[project.scripts] connex-watch = "connex_watch.cli:main"` now (CLI arrives in Stage 02).
2. `[tool.pytest.ini_options]`: `testpaths = ["tests"]`, `markers = ["network: hits the live site"]`,
   `addopts = "-q"`.
3. `[tool.ruff]`: `line-length = 100`, `target-version = "py312"`, lint rules `E,F,I,UP,B`.
4. `test_smoke.py`: asserts `connex_watch.__version__ == "0.1.0"` and that importing the package does
   not import `httpx` at module import time (a cheap guard against import-time network/config).

**TDD note:** here the "test first" is trivial; the real check is that the runner works.

**Run:**

```bash
cd /home/jeka/DeepseekBench
python3 -m venv .venv && . .venv/bin/activate
pip install -e "pipeline[dev]"
pytest pipeline -q            # 1 passed
ruff check pipeline
```

Commit: `stage-00: python package skeleton`.

---

## Task 0.3 — Fixtures and loaders

**Files:** `pipeline/tests/fixtures/*` (copied), `pipeline/tests/fixtures/README.md`,
`pipeline/tests/conftest.py`.

Copy from `plan/assets/fixtures/` (already captured 2026-11-01, see `01-discovery.md`):

| Source | Destination | Contents | Facts to assert later |
|---|---|---|---|
| `search-dev-p1.html` | `fixtures/search-dev-p1.html` | DEV query, page 1 | 16 cards · 4 available / 5 check / 7 not · pager `1 Of 2` · first unit `CX-84121`, ฿45,000, 65 sqm, 1 bed, 1 bath, floor 27, `Sa Thon`, `Bangkok`, updated `2026-10-29` |
| `search-dev-p2.html` | `fixtures/search-dev-p2.html` | DEV query, page 2 | 16 cards · 0 available · pager `2 Of 2` |
| `search-boundary-p71.html` | `fixtures/search-boundary-p71.html` | FULL query, page 71 | 12 available + 4 check · pager `71 Of 546` |
| `search-error-out-of-range.html` | `fixtures/search-error-out-of-range.html` | page beyond last | 105 bytes, JS redirect to `/error/th`, 0 cards |
| `detail-cx84121.html` | `fixtures/detail-cx84121.html` | unit `CX-84121` | `center: {lat: 13.7187616, lng: 100.5202356}`, project `p00348`, breadcrumb name `The Bangkok Sathorn` |

`fixtures/README.md` records: capture date, exact URLs, HTTP status, byte sizes, and the rule
"fixtures are frozen; do not re-capture to make a test pass — fix the parser or the assertion".

`conftest.py` exposes pytest fixtures that read the file as `str` **with `encoding="utf-8"`**:
`search_dev_p1`, `search_dev_p2`, `search_boundary_p71`, `search_error_page`, `detail_cx84121`.
No parsing logic in conftest.

**Verify:** `pytest pipeline -q` (still 1 test) and `python -c "…"` reading each fixture (or a
throwaway test) — all five load, byte sizes match the README table.

Commit: `stage-00: add html fixtures and loaders`.

---

## Task 0.4 — Makefile: the only entry point anyone should need

**Files:** `Makefile`.

Targets (exact names, later stages append):

| Target | Runs |
|---|---|
| `test` | `pytest pipeline` + `pnpm -C web test --run` (web part added in Stage 05; guard with `[ -d web ]`) |
| `check` | `ruff check pipeline` + `pytest pipeline` + `pnpm -C web exec tsc --noEmit` + `pnpm -C web test --run` |
| `crawl` | `connex-watch crawl --preset daily` (added Stage 02) |
| `build` | `connex-watch build …` (added Stage 03/04) |
| `publish` | copy `data/runs/<today>/dataset.json` → `web/public/data/dataset.json` (Stage 05) |
| `nightly` | `scripts/nightly.sh` (Stage 08) |

`make check` must be runnable from the repo root and exit non-zero on any failure.

**Verify:** `make check` green (web parts skipped gracefully).

Commit: `stage-00: add make targets`.

---

## Acceptance criteria

- [ ] `git log --oneline` shows 4 commits, all prefixed `stage-00:`
- [ ] `pytest pipeline -q` → 1 passed
- [ ] `make check` → exit 0
- [ ] `ls pipeline/tests/fixtures` → 5 html files + README
- [ ] `grep -c "connex.in.th" pipeline/tests/*.py` → 0 (no network in tests)
- [ ] repo tree matches `plan/02-architecture.md` §2 (minus files marked for later stages)

## Result (fill after execution)

| Task | Test(s) | Status |
|---|---|---|
| 0.1 | — | |
| 0.2 | `test_smoke.py::test_version` | |
| 0.3 | fixture load smoke | |
| 0.4 | `make check` | |

## Traps

- Do **not** `pip install` inside `pipeline/`; the venv lives at the repo root.
- `pytest` must be run from the repo root (`pytest pipeline`) or with `pythonpath` set; keep it
  simple by documenting the root invocation.
- Fixtures are UTF-8 files with a BOM-free start; always pass `encoding="utf-8"` explicitly on
  Windows-hostile setups.
- Keep `plan/` out of any packaging.
