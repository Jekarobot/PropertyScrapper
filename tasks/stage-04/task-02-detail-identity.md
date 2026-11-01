---
id: 04-02
stage: 04
title: Project code, breadcrumb name and unit code
points: 1
owner: pipeline
status: todo
depends_on: [04-01]
files:
  - pipeline/src/connex_watch/detail_parser.py
  - pipeline/tests/test_detail_parser.py
---

# 04-02 · Project code, breadcrumb name and unit code

**Points:** 1 · **Stage:** 04 — Enrichment · **Owner lane:** pipeline · **Depends on:** 04-01 · **Unblocks:** 04-05

## Goal

Turn the three identity fields into rules with negative cases: the project code is scoped to the breadcrumb, the canonical name is the breadcrumb link text, and the unit code is the token in the last breadcrumb segment.

## Why this task exists

Stage 03 groups units by free-text name, so two spellings of one project become two buildings. The detail page is the only place that carries the **canonical** project code (`p00348`) and the canonical building name, and those two values are what stage 04 later merges on (task 04-12) and what becomes `building.id` in the dataset. A naive implementation reads the first `project_code=` on the page — this fixture also links `s00053` (a train station) and `z00015` (a zone), and a generic scan happily returns one of those. The URL slug is tempting too, but it is SEO copy with the unit code glued on; the breadcrumb is the contract.

## Current state (preconditions)

- 04-01 done: `DetailInfo` with six fields and `parse_detail_page(html)` exist; coordinates and a straightforward identity read are in place; all seven cases of 04-01 are green.
- `pipeline/tests/fixtures/detail-cx84121.html` is unchanged: `.map-nav` at the top of `<main>`, the project anchor, the trailing segment text ending in `CX-84121`, the `<h1>` SEO title, and a canonical link carrying the URL slug.
- `pytest pipeline -q` is green.

## Read before you start

- `CLAUDE.md` §3 and §6.
- `plan/01-discovery.md` §7 — the breadcrumb table (three rows: project code, canonical name, unit code) and the "attributes that look like data" warning in §9.
- `plan/02-architecture.md` §3.1 — `Building` identity is `project_code` when known, else the slug; §6 rule P7. `plan/stage-04-enrichment.md` Task 4.4 — why the code, not the title, is the merge key.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/detail_parser.py` | scoped identity extraction and reason strings |
| modify | `pipeline/tests/test_detail_parser.py` | the six new cases, existing cases untouched |

Nothing else may change: no cache, no enrich module, no fixture edits.

## Test first (red)

Add these cases to the existing file. Synthetic pages are the frozen fixture with one region rewritten in the test — no new committed HTML.

1. `test_detail_parser.py::test_project_code_is_scoped_to_map_nav` — a decoy anchor carrying `project_code=z99999` inserted before `<main>` → `project_code == "p00348"`, not `z99999`, not `s00053`.
2. `...::test_building_name_is_the_breadcrumb_link_text` — frozen fixture → `"The Bangkok Sathorn"`, explicitly **not** the `<h1>` text `"RENT Condo The Bangkok Sathorn Fully furnished BTS-Surasak"`.
3. `...::test_unit_code_is_the_last_breadcrumb_segment` — the segment's trailing `CX-84121` rewritten to `CX-99999` → `unit_code == "CX-99999"`, proving the code is read from the segment and not from the `<h1>` unit block.
4. `...::test_missing_project_link_yields_none_and_reason` — the project anchor removed from `.map-nav` → `project_code is None`, `building_name is None`, `reason == "no-project-code"`, and coordinates are still parsed.
5. `...::test_url_slug_is_never_used_for_identity` — `.map-nav` removed while the canonical link still contains `…-the-bangkok-sathorn-…-cx-84121` → `building_name is None`, `unit_code is None`, `reason == "no-breadcrumb"`.
6. `...::test_unit_code_requires_uppercase_cx_and_digits` — segment text ending in `cx-84121` (lowercase) or `CX-` (no digits) → `unit_code is None`, `reason == "no-unit-code"`.

Expected red: `AssertionError` on cases 1–3 and 6 (`assert 'z99999' == 'p00348'` style) and on the two reason strings (`assert None == 'no-project-code'`) — 04-01 never set a reason.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_detail_parser.py -q`

## Implement (green)

- Keep the public contract from 04-01: `parse_detail_page(html) -> DetailInfo`, same six fields, no state.
- Project code: the first `project_code=<letter><digits>` match **inside** the `.map-nav` block only. Never search the whole document; the head/nav/related-project blocks carry other codes.
- Canonical name: stripped text of the `.map-nav` anchor whose link contains `project_code=`. Never the `<h1>`, never the `<title>`, never the canonical/`og:url` slug.
- Unit code: first `CX-\d+` token in the **last** `/`-separated segment of the `.map-nav` text. Case-sensitive, digits required.
- `reason` is one string describing the first defect found, in this fixed order: no `.map-nav` at all → `"no-breadcrumb"`; no project anchor → `"no-project-code"`; no unit-code token → `"no-unit-code"`; otherwise None. Coordinates are unaffected by identity defects. Task 04-03 prepends coordinate reasons to this order and owns the combined precedence.
- Defects are recorded, never raised: a page with a broken breadcrumb still returns a `DetailInfo` so the enrichment loop can keep going.
- One-sentence docstring update on the parser stating what it does when the breadcrumb is missing.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_detail_parser.py -q` — thirteen green.

## Refactor / keep it DRY

The `.map-nav` block lookup and the segment split must each exist once, as private helpers used by all three fields; 04-03 reuses them. The reason strings are module constants, not literals sprinkled in branches, so 04-03 can extend the vocabulary in one place. No second copy of the selector.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_detail_parser.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Expected: 13 passed, whole suite green, `ruff` exits 0. Manual check: `grep -c "project_code="` on the fixture prints dozens — that is exactly why the search is scoped.

## Out of scope (do not do)

- Coordinate reasons and the truncated-HTML case — 04-03. The cache, loop, merge, CLI — 04-05 onward.
- Parsing the URL slug, the `<title>`, `og:url`, or the `<h1>` as a fallback for any field.
- Thai-market breadcrumb variants, multi-breadcrumb layouts and empty-link-text corners: not tested, not coded. Changing `DetailInfo`'s fields or the coordinate rules.

## Definition of done

- [ ] All six new cases exist, named as written, and pass; the seven 04-01 cases still pass.
- [ ] Red run observed and one failure line quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] Only the two files above plus this task file are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled with commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message (first line exactly, then body):

```text
04-02: scope detail identity to the breadcrumb

- points: 1
- red: test_project_code_is_scoped_to_map_nav failed (assert 'z99999' == 'p00348')
- read project code, name and unit code only inside .map-nav
- record no-breadcrumb/no-project-code/no-unit-code reasons
- suite: N passed
```

Files in the commit: the two files above and this task file (status + Result). Nothing else.

## Traps and review notes

- The fixture has many `project_code=` links; a whole-document scan passes 04-01 by luck and fails case 1. Grep the implementation for a bare `project_code=` search.
- Do not derive `building_name` from the slug or the `<h1>`: both look right on this fixture and both are wrong on the next one.
- `CX-\d+` must not match `cx-`, `CX-` with no digits, or a code embedded in a longer token.
- Reviewers: `grep -n "map-nav" ` should show one block lookup; `grep -n "h1\|canonical"` in the parser should show nothing.

## Plan samples are untrusted

The plan gives the three extraction rules as regex illustrations. Verify each against the frozen fixture and the installed BeautifulSoup version (`CLAUDE.md` §6) before copying; record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
