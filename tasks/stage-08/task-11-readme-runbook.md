---
id: 08-11
stage: 08
title: Runbook, exit codes and troubleshooting
points: 1
owner: any
status: todo
depends_on: [08-03, 08-05]
files:
  - README.md
  - docs/screenshot.png
---

# 08-11 · Runbook, exit codes and troubleshooting

**Points:** 1 · **Stage:** 08 — Hardening and release · **Owner lane:** any (docs) · **Depends on:** 08-03, 08-05 · **Unblocks:** 08-12

## Goal

Fill the seven README sections with a runbook a stranger can follow: what the product is with a screenshot from a real run, how to start it, how fresh the data is, what each exit code means, the six failures we expect, the non-goals, and the
ethics paragraph.

## Why this task exists

The repository now has two programs, a scheduler, a dataset pipeline and exit codes that live only in commit messages and plan files. The person who needs this most is the owner at 02:40 when the timer produced nothing: one page that says what
to run, what the failure code means and what to do, without reading `tasks/` or `plan/`. The seven sections are already stubs from 00-02, so this task fills them in place. The troubleshooting list is deliberately limited to the six failures the
stages actually produced, because a generic FAQ teaches nothing, and it pins the code-to-documentation interface that 08-12 re-reads during the release pass.

## Current state (preconditions)

- 00-02 is done: `README.md` line 1 is `# Connex Watch`, then a two-sentence description and exactly seven `##` sections in order: What it is, Quick start, Data freshness, Exit codes, Troubleshooting, Roadmap, Ethics — each a placeholder.
- 08-03 is done (the nightly prunes and rotates, so the disk story is truthful) and 08-05 is done ("Data freshness" already carries the unit install commands, the journal hint and the cron line).
- 08-06 is done: `make check`, `make nightly`, `make publish` and `make help` behave as documented.
- 02-14, 03-14 and 04-15 define the codes: 0 success, 1 data/check failure, 2 usage or unreadable input, 3 blocked by the site, 4 parse error.
- `docs/` does not exist; a real dataset can be produced locally and served with `pnpm -C web dev`. Nothing here touches the network.

## Read before you start

- `CLAUDE.md` §5 (no invented features in the roadmap), §6, §7, §8, §10 (no personal data in a screenshot).
- `plan/stage-08-hardening-ops.md` Task 8.5 — the seven sections, the six failures, the screenshot rule.
- `plan/02-architecture.md` §9 — the seven non-goals, one line each; §3.3 for `generatedAt` and `stats.partial`.
- `plan/90-test-matrix.md` "Manual acceptance checklist"; `README.md` as it stands; the `Makefile` targets; the two unit files; `cli.py` for the codes.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `README.md` | the seven sections, 3–10 lines each; no new section |
| create | `docs/screenshot.png` | one capture of the running app on the real published dataset |

Nothing else: no new section, no `docs/*.md`, no second image, no `Makefile`, no code.

## Test first (red)

Documentation scaffolding: the checks below are the evidence (`CLAUDE.md` §3 exception). Run them before and after; record the "before" line of checks 1 and 4.

1. `check-1` `grep -n '^## ' README.md` — exactly seven headings in the order above. Before: seven headings with placeholder text; after: the same seven, filled.
2. `check-2` section length — for each section, count the non-empty lines between its heading and the next; every count is between 3 and 10. One line or twenty fails.
3. `check-3` exit codes — the Exit codes section holds a table with the rows 0, 1, 2, 3, 4, each with a meaning matching `cli.py`: 0 success, 1 data check failed / nothing published, 2 usage or unreadable input, 3 blocked by the site, 4 parse
   error. No fifth code, no "5 = unknown".
4. `check-4` troubleshooting — the section names all six failures in the plan's words: `0 cards on page 1`, `blocked (403)`, `cache corrupt`, `dataset older than 48 h`, `many buildings without coords`, `map is grey`; each has a one-line cause
   and a one-line action. Before: the placeholder sentence.
5. `check-5` roadmap — exactly seven numbered entries matching `plan/02-architecture.md` §9 in substance (cross-run history, sale listings, backend or server-side filtering, accounts and notifications, photos and descriptions, multi-city
   presets, i18n), nothing reading like a promise to build them, no "coming soon".
6. `check-6` screenshot — `docs/screenshot.png` exists, `file` reports PNG, it is under 1 MB, and "What it is" references it as `docs/screenshot.png`. Visual review: no browser chrome, bookmarks bar, login state or devtools, no personal data;
   the capture shows the local `pnpm dev` page with the real published dataset.
7. `check-7` no code and no secrets — `grep -c '^```' README.md` is `0` (commands appear as inline code spans only), and a grep for secret-shaped strings (`api[_-]?key`, `token=`, `Bearer `, `sk-`, `AKIA`, `password`) finds nothing.

Expected red: check 2 fails on placeholder-length sections, check 4 finds no failure names, check 6 finds no PNG. Quote check 4's `grep -c` output before the change.

## Implement (green)

- **What it is** (3–6 lines): keep 00-02's two sentences — a scheduled pipeline keeps only `available` units from the public rent listings and publishes one static `dataset.json`; the viewer is a Vite + React + Leaflet SPA, no backend. Then the screenshot line with a one-sentence caption saying it came from a real run.
- **Quick start** (3–8 lines): the three entry points — `make check` proves the toolchain offline, `make nightly` refreshes the data, `pnpm -C web dev` serves the app — plus the prerequisites (venv at the repo root, pnpm) and a note that `make check` passes on a fresh clone.
- **Data freshness** (5–10 lines): where `generatedAt` comes from (the build clock, local time), what `stats.partial` means (enrichment stopped early; the web app shows a banner), how to force a full crawl (`connex-watch crawl --preset daily
  --full`), the 48 h staleness rule `check` enforces, and 08-05's schedule paragraph with the install commands, the journal command and the cron alternative — keep that text, do not rewrite it.
- **Exit codes** (4–8 lines): the five codes with one clause each and, for 1 and 3, the action ("nothing was published — read the log"; "wait and re-run later").
- **Troubleshooting** (8–10 lines): the six failures, each a bold lead-in with cause and action. Fixture re-capture needs the owner's approval; "blocked" must say do not change the User-Agent; the cache entry names `data/cache/buildings.json` and the ~20-minute cold rebuild.
- **Roadmap** (4–8 lines): the seven non-goals as a numbered list, one line each, opening with a sentence that v1 is feature-frozen and these are future work, not commitments.
- **Ethics** (3–5 lines, a paragraph): the politeness floor (a second between requests, backoff on errors), robots.txt allows the paths used, the site is read-only for us (no POST, no login, no form), and no personal data is stored — no phone numbers, no agent names, no contact-page content; name the crawler's User-Agent and why it identifies the tool.
- Everything else in `README.md` stays: the title, the two sentences, the seven headings and their order. No badges, licence section, emoji or marketing.

Run: checks 1–7 again — all pass.

## Refactor / keep it DRY

The seven headings exist once (00-02 created them) and this task never adds an eighth; the schedule paragraph lives in Data freshness only; the exit codes appear in the README and in the CLIs' behaviour, and the README is the prose version —
summarise the meaning, do not paste the CLI's messages. One screenshot, referenced once.

## Verification (how to test it)

```text
grep -n '^## ' README.md
grep -c '^```' README.md
grep -n 'cache corrupt\|map is grey\|blocked (403)' README.md
file docs/screenshot.png
make check
```

Expected: seven headings in the documented order; `0` fences; each grep finds its lines; a PNG; `make check` green. Manual read: hand the README to someone who has not seen the repo and ask them to start the app and explain exit code 3 — they
should need nothing else.

## Out of scope (do not do)

- A new README section (contributing, architecture, licence, changelog, API), badges, a TOC, emoji, or a translated README.
- Copying plan text, pasting CLI help output, or documenting a flag that does not exist — run `make help` and `connex-watch --help` first.
- More than one screenshot, an animated GIF, a diagram, or a screenshot from the live site (third-party content, possible personal data).
- Documenting the backlog as planned work, promising dates, or describing a non-goal as implemented.
- Any code change, dependency, or edit to `plan/`, `tasks/`, `Makefile` or `scripts/`.

## Definition of done

- [ ] Checks 1–7 pass; every section is 3–10 non-empty lines.
- [ ] Before/after evidence quoted in the commit body (missing failure names, missing PNG).
- [ ] The screenshot is from a real run, with no browser chrome, login state or personal data.
- [ ] `make check` green; only `README.md`, `docs/screenshot.png` and this task file changed.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and the capture facts (date, units, buildings).
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
08-11: write the runbook in the README

- points: 1
- red: grep found no 'cache corrupt' and no map-is-grey entry; docs/screenshot.png missing
- fill the seven sections: what it is, quick start, freshness, exit codes, troubleshooting, roadmap, ethics
- add one screenshot captured from a real published dataset
- suite: 17 nightly cases + 6 harness modes, make check green
```

Files in the commit: `README.md`, `docs/screenshot.png`, this task file. One commit, on trunk `main`.

## Traps and review notes

- Writing prose first and checking the numbers later is how a README documents a flag that was renamed: run `make check`, `make help` and each CLI's `--help` before writing a command line.
- A screenshot from the live site carries third-party markup, ads and possibly personal data; the capture must be the local page with the published dataset.
- "Blocked" advice suggesting a User-Agent change or a lower delay contradicts `plan/02-architecture.md` §8 and `CLAUDE.md` §10; the only correct action is to stop and wait.
- Documenting the roadmap as "planned for v2" turns a YAGNI statement into a promise; keep §9's non-committal tone.
- Reviewers: seven `##` headings, zero fences, and every command in the README must be one you ran in this task.

## Plan samples are untrusted

The plan names the six failures and the seven sections, but its commands predate the shipped `Makefile` and CLI (`make nightly` goes through `scripts/nightly.sh`, and `check` has a dataset step). Verify every command by running it, keep 08-05's
scheduling text unchanged, and record the deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |

