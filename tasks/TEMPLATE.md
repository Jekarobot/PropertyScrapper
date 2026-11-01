# Task template — copy this structure exactly

Every file in `tasks/stage-NN/` uses this skeleton, in this order. Delete the guidance in angle
brackets and fill everything else. Target length: 80–160 lines (density beats brevity; the reference task is longer). No code blocks, no source snippets
(see `CLAUDE.md` §7).

````markdown
---
id: NN-MM
stage: NN
title: <imperative-ish noun phrase, <= 60 chars>
points: <0.5 | 1 | 2 | 3>
owner: pipeline | web | any
status: todo
depends_on: [NN-MM, ...]
files:
  - <path relative to repo root, one per line>
---

# NN-MM · <Title>

**Points:** <n> · **Stage:** NN — <stage name> · **Owner lane:** pipeline|web · **Depends on:**
<task ids or "nothing"> · **Unblocks:** <task ids>

## Goal

One sentence: the single thing this task adds or fixes. If you cannot say it in one sentence, the
task is miscut.

## Why this task exists

3–6 sentences for an engineer with zero context: where this piece sits in the pipeline or the UI,
what breaks or stays impossible without it, and what depends on it later. Name the product-level
behaviour, not the implementation.

## Current state (preconditions)

What must already exist when you start. Name the files/functions that must be present and the task
that created them. If one is missing, stop: the previous task is not done.

## Read before you start

- `CLAUDE.md` §<relevant sections> — <why>
- `plan/<doc>.md` §<n> — <what to extract>
- <existing source file> — <what to look at>
- <fixture / data file> — <facts to rely on>

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create/modify | `<path>` | <one line> |
| create | `<test path>` | <cases> |

Nothing else may change in this task. If you believe another file is required, stop and report.

## Test first (red)

Write these tests **before** any production code. For each case: the test name, the input, and the
expected result. Use exact values from the plan/fixtures (never approximate assertions).

1. `<test_file>::<test_name>` — <input> → <expected>. <Why it matters / which quirk it pins.>
2. ...

Expected red symptom: <e.g. "ImportError: cannot import name X" / "AttributeError: module has no
attribute Y" / "assert 0 == 16">. If it fails differently, fix the test or the assumption first.

Run: `<exact command>` — confirm every new case is red for the stated reason.

## Implement (green)

Describe the behaviour to implement, not the code. Cover:

- the public contract: function/component names, parameters, return type, error type;
- the rules and edge cases (ordering, ranges, nullability, defaults);
- where the logic lives (module) and what it must **not** do;
- docstring/comment expectation for public entry points.

Keep it minimal: only what the listed tests demand. No speculative options, no extra fields.

Run: `<exact command>` — all new cases green.

## Refactor / keep it DRY

Name the duplication this task could introduce and where the single implementation must live.
Delete any temporary duplication before committing. Re-run the suite after refactoring.

## Verification (how to test it)

```text
<command 1 — target tests>
<command 2 — whole lane suite>
<command 3 — lint/typecheck, if applicable>
```

Manual check (if any): <what to run/open and what you must see>. State the exact expected output.

## Out of scope (do not do)

- <thing a reasonable engineer might add but must not>
- <adjacent field/feature reserved for another task>

## Definition of done

- [ ] Every case under "Test first" exists, is named as written, and passes.
- [ ] Red run observed and quoted in the commit body.
- [ ] Target tests + lane suite green; linter/typecheck clean.
- [ ] Only the files in the table above are modified (`git status --short` checked).
- [ ] Task frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
NN-MM: <imperative summary>

- points: <n>
- red: <test id> failed (<failure line>)
- <what changed, 1–3 bullets>
- suite: <N> passed
```

Files in the commit: exactly those listed in "Files to touch".

## Traps and review notes

- <the concrete mistake this task invites — e.g. splitting HTML on a wrapper, using type=number,
  trusting a cached range, reading state from a detail page>
- <what a reviewer should grep/assert to catch a weak implementation>

## Plan samples are untrusted

Any snippet in `plan/` is illustrative only. Before using one, apply the checklist in `CLAUDE.md` §6
(check pinned versions, the installed API, and the repo's current code). If a sample conflicts with
this repository or the library version, follow the repo/library and record the deviation under
`## Decisions`. Do not add a dependency because a sample mentions one.

## Decisions

<!-- append during implementation; leave empty at planning time -->

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
````

## Notes for the agent writing task files

- The task must be executable by a fresh agent that has read only `CLAUDE.md` and this one file.
  Reference `plan/` sections for depth, but state the facts the task depends on inline.
- Use exact numbers that are already proven: fixture counts (`4/5/7`, `0/1/15`, `12/4/0`), pager
  values (`1 Of 2`, `2 Of 2`, `71 Of 546`), unit `CX-84121` at ฿45 000 / 65 sqm / 1 bed / 1 bath /
  floor 27, `Sa Thon` / `Bangkok`, date `29/10/2569` → `2026-10-29`, coords 13.7187616 /
  100.5202356, project `p00348`. Do not invent new facts.
- Test case names go in the doc; the implementer writes the file. Never paste the test code.
- If the plan is ambiguous, describe the ambiguity under "Traps" and mark the decision the
  implementation must make; do not silently choose an unverified behaviour.
