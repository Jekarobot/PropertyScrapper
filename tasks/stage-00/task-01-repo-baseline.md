---
id: 00-01
stage: 00
title: Verify trunk state, add ignore rules and data/.gitkeep
points: 0.5
owner: pipeline
status: todo
depends_on: []
files:
  - .gitignore
  - data/.gitkeep
---

# 00-01 · Verify trunk state, add ignore rules and data/.gitkeep

**Points:** 0.5 · **Stage:** 00 — Bootstrap · **Owner lane:** pipeline · **Depends on:** nothing · **Unblocks:** 00-02, 00-03

## Goal

Verify the trunk — do not re-create it — and add the ignore rules plus the tracked `data/` placeholder.

## Why this task exists

The repository already exists, is connected to `origin` and holds the plan; `git init`, a second remote or a
force-push would destroy it. Every later run writes under `data/`, so one ignore list keeps `git status` readable
and the "only the listed files" commit rule enforceable, while `data/.gitkeep` keeps the directory in the tree.

## Current state (preconditions)

- Repo root `/home/jeka/DeepseekBench`; `origin` has the fetch and push URL
  `git@github.com:Jekarobot/PropertyScrapper.git`; branch `main`; `git log --oneline` has the plan commits and `git ls-files plan` is non-empty.
- `.gitignore` and `data/` do not exist; the tree is clean apart from untracked `CLAUDE.md` / `tasks/`, which are the owner's baseline, not yours (Traps).

## Read before you start

- `CLAUDE.md` §1 (trunk `main`, one commit per task, no branches, never force-push), §2, §8, §11 — workflow,
  global done, the layout and the commands this repository treats as canonical.
- `plan/stage-00-bootstrap.md` Task 0.1 — the ignore list and "verify, do not re-create"; `plan/02-architecture.md` §2 — what lives under `data/`; `plan/README.md` §2 — the superseded branch rule.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `.gitignore` | exactly the fifteen ignore rules listed below |
| create | `data/.gitkeep` | empty file that keeps `data/` tracked |

Nothing else may change; if another file seems required, stop and report.

## Test first (red)

No pytest file here: pytest arrives with 00-03, and this is the scaffolding case in `CLAUDE.md` §3 where the only test is "the tooling behaves".

1. `check-1 git check-ignore -v .venv/` — today exits `1` and prints nothing; after, it prints the rule and exits `0`.
2. `check-2 git check-ignore -q data/raw/run-x/pages/page-0001.json` — today `1`; after `0`, so a run never shows up in `git status`.
3. `check-3 test -f data/.gitkeep` — today `1`; after `0`, and the file is empty.
4. `check-4 git check-ignore -q pipeline/tests/fixtures/search-dev-p1.html` — must stay `1`: 00-04 commits five HTML fixtures, so no rule may catch `*.html`.

Expected red: `git check-ignore -v .venv/; echo $?` and `test -f data/.gitkeep; echo $?` both print `1` from the repo
root (no venv yet), with no output for the generated paths.

## Implement (green)

Create `.gitignore` with exactly these fifteen entries, one per line, in this order, and nothing else: `.venv/`,
`__pycache__/`, `*.pyc`, `.pytest_cache/`, `.ruff_cache/`, `node_modules/`, `web/dist/`, `web/playwright-report/`,
`web/test-results/`, `web/public/data/`, `data/raw/`, `data/runs/`, `data/cache/`, `.probe/`, `logs/`.

It ignores the three generated `data/` subdirectories, not `data/` itself, so `data/.gitkeep` stays tracked without
`git add -f`; no bare `data/` rule and no negation. Create `data/.gitkeep` empty — no content, no meaning beyond
keeping the directory. No `.gitattributes`, `.editorconfig`, license or rule for a tool that does not exist yet.
Constraints: no test or command touches the network, the fixtures under `pipeline/tests/fixtures/` are frozen (they
arrive in 00-04; never edit or re-capture one), and only the files in the table above are committed.

## Refactor / keep it DRY

`.gitignore` is the only home of the ignore rules: no copy in a script, in `.git/info/exclude` or in a later task.

## Verification (how to test it)

```text
git remote -v
git check-ignore -v .venv/ data/raw/x/y.json data/runs/2026-11-01/dataset.json logs/nightly.log web/public/data/dataset.json
git ls-files data
```

Expected: two `origin` lines with the SSH URL; `git check-ignore -v` prints five rules, one per path; `git ls-files data`
prints only `data/.gitkeep`; after committing, `git status` must not mention `data/` and `git ls-files plan` stays non-empty.

## Out of scope (do not do)

- `git init`, a second remote, a new branch, a rename of `main`, any push or history rewrite —
  `CLAUDE.md` §1 overrides `plan/stage-00-bootstrap.md` Task 0.1 steps 2 and 5 (see Traps).
- `README.md` (00-02), `pipeline/**` and `.venv` (00-03), fixtures (00-04), `Makefile` (00-06).
- CI, pre-commit hooks, `.gitattributes`, `.editorconfig`, `LICENSE`, `.github/`; deleting `plan/`; ignoring `*.html`,
  `web/` or `pipeline/`; committing `CLAUDE.md` or `tasks/` while untracked.

## Definition of done

- [ ] Checks 1–4 behave as written, including the fixture check staying at exit `1`; red quoted in the commit body.
- [ ] Only `.gitignore` and `data/.gitkeep` are modified (`git status --short` checked); `status: done`; `## Result`
      has the commit hash and check evidence; exactly one commit on the trunk with the message below.

## Commit

```text
00-01: add the ignore rules and keep the data directory

- points: 0.5
- red: git check-ignore -v .venv/ exited 1 (no rule matched); data/.gitkeep was missing
- add the fifteen ignore rules and an empty data/.gitkeep
- suite: n/a — pytest is configured in 00-03; checks 1-4 green
```

Files in the commit: exactly `.gitignore`, `data/.gitkeep` and this task file (status and `## Result`). Committed alone on the trunk (`main`): one task = one commit; do not push or branch.

## Traps and review notes

- Do **not** create a `stage-00` branch or push one: `plan/stage-00-bootstrap.md` Task 0.1 still says "work on a
  branch", `CLAUDE.md` §1 supersedes it, and that plan text is stale.
- No bare `data/` rule (it hides `data/.gitkeep`) and no `*.html` rule: 00-04 commits five fixtures. Do not fake a
  clean tree by committing `CLAUDE.md` / `tasks/` or deleting `plan/`; report instead.
- Reviewer: `git check-ignore -v` per generated path, `git ls-files data`, and `grep -n 'html\|web/\|pipeline/' .gitignore`.

## Plan samples are untrusted

The plan states the ignore rules in a fenced block and gives branch steps that are now stale: apply `CLAUDE.md` §6,
check `git --version`, the actual branch and remote, and follow this repository over the plan.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
