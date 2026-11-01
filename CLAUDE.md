# CLAUDE.md — working agreement for Connex Watch

This file is the contract for every agent (human or AI) that implements the project. Read it fully
before the first task. The tasks in `tasks/` assume you have **zero** context about this repository;
this file and the task file together are the whole brief.

Design sources of truth (read-only, never edit without the owner's approval):

| Document | What it answers |
|---|---|
| `plan/README.md` | what the product is, stage map, conventions |
| `plan/01-discovery.md` | the connex.in.th site contract: URLs, selectors, quirks, risks |
| `plan/02-architecture.md` | repo layout, data model + JSON schemas, stack, UI rules F1–F11, pipeline rules P1–P10 |
| `plan/90-test-matrix.md` | requirement → test → command (proof of done) |
| `plan/91-agent-briefs.md` | ownership matrix, prompt templates, merge protocol (**branch rules are overridden here**) |
| `plan/assets/fixtures/` | frozen HTML captured 2026-11-01 |
| `tasks/` | the executable breakdown: one file = one task = one commit |

If a task contradicts `plan/01-discovery.md` or `plan/02-architecture.md`, the task wins for *scope*
and the architecture doc wins for *data shape*. Record the conflict in the task file's "Decisions"
section (see §9) and continue. If the conflict changes a public interface, stop and ask the owner.

---

## 1. Repository, trunk and the one-commit rule

- Repository root: `/home/jeka/DeepseekBench`. Remote `origin`:
  `git@github.com:Jekarobot/PropertyScrapper.git` (SSH, owner's key).
- **Trunk = the default branch, currently named `main`.** The owner calls it "master"; they are the
  same thing. Do **not** create a `master` branch, and do **not** create `stage-NN` branches.
- **Work one task at a time and commit each task to the trunk.** No feature branches, no topic
  branches, no merge commits. This overrides `plan/91-agent-briefs.md` §5 and `plan/README.md` §2.
- Never force-push, never rewrite published history, never amend a commit that has been pushed.
- Never commit generated artifacts: `data/`, `web/public/data/`, `node_modules/`, `.venv/`,
  `web/dist/`, `logs/`, `web/test-results/`, `web/playwright-report/`, `.probe/`.
- Before committing: `git status --short` must list **only** the files named in the task file. If
  anything else is dirty, stop and investigate — do not "clean up" unrelated files into your commit.

Commit message shape (exact prefix `NN-MM:` from the task id):

```
01-05: discover result cards on a search page

- points: 2
- red: pytest ... test_card_discovery failed (ImportError: parse_search_page)
- add card discovery and position tracking
- suite: 41 passed
```

One task = exactly one commit. If a task turns out to need two independent changes, its size was
wrong: stop, report, and split it into a new task file (with the owner's approval) rather than
making a fat commit.

---

## 2. Task workflow (the only loop)

1. **Read** the task file completely, then the documents it lists under "Read before you start".
   Do not start editing before you have read `plan/01-discovery.md` at least once in this session.
2. **Check the preconditions** listed in the task ("Current state"). If they are not true, the
   previous task is not done: fix that first or report a blocker.
3. **Set status:** change the task frontmatter `status: todo` → `status: in-progress`.
4. **Write the failing test first.** The task lists the test file and the cases. Write only those
   cases (plus trivial helpers). Run the exact command from the task and confirm it fails **for the
   reason the task predicts** (missing module, missing function, wrong value). A test that fails with
   a syntax error or a typo is not red, it is broken.
5. **Capture the red evidence:** the test id and one line of the failure. It goes into the commit body.
6. **Implement the minimum** that makes those tests pass. No extra features, no speculative
   parameters, no abstractions for a single caller.
7. **Run the task's tests, then the whole lane suite** (`make check` once it exists; before that,
   `pytest pipeline -q` or `pnpm -C web test --run`). Re-read the task's "Out of scope" list and
   remove anything you added beyond it.
8. **Update documentation only where the task says** (usually the `RESULT` table of the task file,
   optionally a `README.md` section). Do not edit `plan/` unless the task explicitly says to.
9. **Commit** with the task's commit message, only the files the task lists.
10. **Set status:** `status: done` (or `blocked` with a reason) in the frontmatter, and record the
    commit hash and test counts in the task file's `RESULT` section. This bookkeeping belongs to the
    **same** commit as the work (make it one commit by doing step 9 and 10 together; amend only
    before pushing).

No task is "done" because the code looks right. It is done when the task's "Definition of done"
checklist is green, evidence is pasted, and the commit exists.

### Size guard

If a task grows past roughly 1.5× its story points (e.g. a 1-point task needs a new dependency, a
schema change, and a refactor), **stop**. Commit nothing half-done; write the finding in the task
`RESULT` section and report. Oversized tasks get split, never absorbed.

---

## 3. TDD rules (red → green → refactor)

- **Test first, always.** No production code before a failing test exists. Exception: scaffolding
  config files whose only test is "the tool runs" — the task says so explicitly.
- The red run must be *observed* and *quoted*. "It would have failed" is not evidence.
- Never weaken an assertion to make a test pass. Never delete, `skip`, or `xfail` a test to go green.
  If a documented expectation is wrong, prove it with the fixture/reference, report it, and fix the
  *document* in the same commit.
- Test names state the behaviour: `test_zero_cards_inside_range_raises_parse_error`, not `test_2`.
- One behaviour per test. Table-driven tests are fine for pure functions (parsers, filters, format).
- **Fixtures are frozen.** `pipeline/tests/fixtures/*.html` and
  `web/src/test/fixtures/dataset.sample.json` are the contract. If a fixture disagrees with a doc,
  stop and report; never re-capture a live page to make a test pass (and never re-capture without the
  owner's approval — see §10).
- **Unit tests never touch the network.** Pipeline: inject a fake fetcher or use a saved fixture. Web:
  `vi.stubGlobal("fetch", …)` or pass a fake `fetchImpl`. The only network tests are marked
  `@pytest.mark.network` and are opt-in.
- **Determinism:** inject the clock (`now=` / `clock=`) and assert byte-identical output across two
  runs (ignoring `generatedAt`). No `datetime.now()` buried in a function under test.
- Refactor after green, with the suite still green. Refactors that change behaviour need their own
  test first.

---

## 4. Story points — what each size means here

| Points | Meaning | TDD evidence expected |
|---|---|---|
| **0.5** | One simple, mechanical change plus its direct test: a constant, a formatter, a button, a one-branch guard, a single selector. | 1–3 focused test cases. |
| **1** | A normal change: a pure function, a small module, a CLI flag, a schema field, a component branch, parsing one more field. | 3–8 cases, negative cases included. |
| **2** | Medium-heavy: a component with real behaviour, an integration of two modules, a state machine, a parser for a whole page, a map layer. | 6–12 cases, at least one integration-level test. |
| **3** | Complex: needs investigation first (library behaviour, browser/Leaflet mechanics, site quirks), *then* implementation. Write down what you learned in the task `RESULT` before coding. | A spike note + 8+ cases + one end-to-end/integration proof. |

"A task is one thing" is the rule behind the numbers: one behaviour, one file cluster, one commit.
If you cannot name the single thing in one sentence, the task is mis-cut.

---

## 5. DRY and YAGNI, enforced

- **DRY:** one implementation of each rule. Filters live in `web/src/lib/filter.ts`, not in components;
  URL building lives in `urls.py`; grouping lives in `build_dataset.py`/`lib/group.ts`; the unit row
  markup lives in `components/UnitRow.tsx` and is reused by the results list and the building panel.
  If you need the same logic twice, extract it and delete the duplicate — do not "keep both for now".
  Duplication of *test data* is fine; duplication of *logic* is not.
- **YAGNI:** no database, no backend, no auth, no router, no state library, no URL-sync of filters, no
  cross-run history, no image gallery, no i18n, no fuzzy name matching, no abstraction with one
  implementation. The backlog in `plan/02-architecture.md` §9 is explicitly *not* to be built.
- **No new dependency** without a one-line justification in the commit body and the owner's approval.
  Everything needed is already chosen in `plan/02-architecture.md` §4. Pin exact versions.
- **Single source of truth:** `App.tsx` composes state once; components are dumb; the pipeline writes
  one dataset file; the web app knows nothing about `connex.in.th` HTML.
- **No `any`** in `web/src/lib/**`, `web/src/state/**`, `web/src/api/**`. No `# type: ignore` in
  `pipeline/src` without a one-line reason.

---

## 6. Code samples inside `plan/` and `tasks/` are UNTRUSTED

The plan was written before the code existed. Its snippets are illustrations, may be stale, may not
compile, may use the wrong version of a library, and may contain bugs. **Never paste a sample from
`plan/` into the repo without verifying it against this repository and the installed versions.**
This is a hard rule, not a stylistic preference.

Verification checklist before you use any sample (function signature, selector, version, command):

1. **Versions:** read the pins — `pipeline/pyproject.toml`, `web/package.json`, the lockfiles. Check
   what is actually installed: `python -c "import bs4, pydantic, httpx; print(bs4.__version__, pydantic.VERSION, httpx.__version__)"`,
   `pnpm -C web list --depth 0`.
2. **API reality:** read the installed source or type definitions (`site-packages/`, `node_modules/`,
   `.d.ts`) for the exact version. Do not rely on memory or on the plan's description of an API.
3. **Repo reality:** read the current module/test the sample touches. Signatures in the repo win over
   signatures in the plan.
4. **Behaviour:** turn the sample's claim into a test case before trusting it. If the sample says
   "`select_one` returns the card", assert it on the real fixture first.
5. **Conflict:** if the sample disagrees with the installed library or the repo, follow the library +
   repo, and write the deviation in the task's `Decisions` section. Never add a dependency only
   because a sample uses it.
6. **Commands in samples are not gospel either:** a planned command may reference a flag or target
   that does not exist yet. Run `make help` / `--help` before documenting it.

Known-good environment facts (verified 2026-11-01 on the owner's machine; re-check, do not assume):
Node `v22.20.0`, pnpm `12.16.1`, Python `3.12.3`, GNU Make `4.3`, git `2.43.0`, branch `main`,
repo root `/home/jeka/DeepseekBench`, fixtures in `plan/assets/fixtures/` with the exact counts listed
in `plan/assets/fixtures/README.md`.

---

## 7. Planning documents contain no code

`tasks/**/*.md` describe **what** to do, never **how** in code. Do not add code blocks, code
snippets, or copy-pasted source to a task file, not even "for clarity". Behaviour is described in
prose, tables and test case names. If a decision needs precision, describe the contract: names of
functions/fields, input → output, error conditions, ordering rules. When you implement, the code lives
in the repo, not in the task.

The same applies to `plan/`: it already contains samples; do not add more, and do not treat existing
ones as authoritative (§6).

---

## 8. Definition of done (global)

A task is done when **all** of these are true:

- [ ] The task's test file exists, and every case listed in the task is present and named.
- [ ] The red run was observed and quoted in the commit body (test id + failure line).
- [ ] The task's target tests pass; the full lane suite passes (`make check` once available).
- [ ] `ruff check pipeline` is clean (pipeline tasks) and `pnpm -C web typecheck` is clean (web tasks).
- [ ] No network in unit tests; no skipped/`xfail`ed tests added; no assertion weakened.
- [ ] Every public function/component added has a one-sentence contract in its docstring/comment.
- [ ] Only the files listed in the task were touched (`git status --short` evidence).
- [ ] The task frontmatter says `status: done`; the task `RESULT` table has the commit hash and test
      counts; any deviation is recorded under `Decisions`.
- [ ] Exactly one commit on the trunk with the task's `NN-MM:` message.

---

## 9. Decisions and deviations — where they go

Every task file ends with two append-only sections:

- `## Decisions` — anything the implementation had to choose that the task did not specify (e.g. "we
  treat a 2-part location line as `district = None` and log a warning"). One bullet per decision,
  with the reason.
- `## Result` — commit hash, test counts, observed red line, measured numbers (runtime, request
  counts), and anything surprising.

Never rewrite the task's requirements to match what you built. If reality forces a scope change, the
owner decides; the task file records it.

---

## 10. Safety, courtesy, and the live site

- **No live crawl, no detail-page fetch, no POST, no form submission** without the owner's explicit
  approval in the session. The pipeline has been designed to be polite; tests must use fixtures.
- The only opt-in network test is the one in `02-16`; it is skipped unless `-m network`. Never let
  `make check` hit the network.
- Never store or log a full detail page (it is ~220 KB and carries third-party markup). Parse, extract
  the numbers, discard.
- The Google Maps key embedded in the site's HTML is **not ours**: read coordinate numbers only, never
  call Google APIs.
- No secrets in the repo. Fixtures are redacted before commit (procedure in
  `plan/assets/fixtures/README.md`); if you ever re-capture one, redact first, then commit.
- Do not install or enable systemd units, cron jobs, or anything that runs unattended. Document them;
  the owner installs them.
- Do not touch personal data: no phone numbers, no agent names, no contact forms.

---

## 11. Environment and commands

```text
repo root      /home/jeka/DeepseekBench
python venv    .venv (created in task 00-03; activate before pipeline work)
pipeline       pytest pipeline -q            (from repo root)
lint           ruff check pipeline
web            pnpm -C web test --run
web types      pnpm -C web typecheck
everything     make check                    (offline; added in task 00-06)
```

- `pytest` is run from the repo root with `pytest pipeline`, never from inside `pipeline/`.
- The venv lives at the repo root (`python3 -m venv .venv`), not inside `pipeline/`.
- Use `source .venv/bin/activate` in the same shell as the commands.
- One writer per path: the pipeline lane owns `pipeline/**`, the web lane owns `web/**`. Never run two
  tasks that touch the same file concurrently. `tasks/README.md` §4 defines the lanes and waves.

---

## 12. Handoff to sub-agents (implementation phase)

Each task file is a complete brief. To hand a task to a fresh agent, give it exactly this:

```text
Repository: /home/jeka/DeepseekBench
Read /home/jeka/DeepseekBench/CLAUDE.md first, then
/home/jeka/DeepseekBench/tasks/<stage>/<task file> in full.
Do only that task. TDD: write the listed tests first, observe the red run, implement, run the
task's verification commands, commit to the trunk with the task's commit message, update the
task's frontmatter and RESULT section in the same commit.
Report: commit hash, test counts, red evidence, deviations.
```

Rules for parallel sub-agents:

- One task per agent, one agent per path. Two agents may run in parallel only if their file lists are
  disjoint and neither depends on the other (the task's `depends_on` is empty or already done).
- A finishing agent leaves the tree clean and the suite green; the next agent assumes that.
- The reviewer agent (read-only) runs after each stage: re-runs the stage's commands, checks the
  test matrix rows, greps for weakened assertions, and files findings as a table. The reviewer never
  edits code.

---

## 13. Progress tracking

Task frontmatter is the tracker. Do not maintain a second checklist. From the repo root:

- count by state: `grep -h '^status:' tasks/*/*.md | sort | uniq -c`
- find open work: `grep -l '^status: todo' tasks/*/*.md`
- points remaining: `grep -h '^points:' tasks/*/*.md` (aggregate as needed)

`tasks/README.md` holds the index (id → file → points → dependencies) and the lane/wave plan. If a
task is added or removed, the index and the wave plan change in the same commit as the task file.
