---
id: 01-03
stage: 01
title: "`SearchSpec` defaults and `from_url`"
points: 1
owner: pipeline
status: todo
depends_on: [01-02]
files:
  - pipeline/src/connex_watch/models.py
  - pipeline/src/connex_watch/urls.py
  - pipeline/tests/test_urls.py
---

# 01-03 · `SearchSpec` defaults and `from_url`

**Points:** 1 · **Stage:** 01 — Search-page parser · **Owner lane:** pipeline · **Depends on:** 01-02 · **Unblocks:** 01-04

## Goal

Turn a pasted `search_en` URL into a validated, immutable `SearchSpec`, keeping parameters this code does not understand instead of dropping them.

## Why this task exists

The only way the crawler learns which search to crawl is the URL a human copies out of the browser, so reading it must be lossless enough that the same spec can later be re-rendered, compared with a preset and stored in the run manifest. If parsing silently drops `status=1`, turns `bedroom=` into 0 or `sort=` into absent, the generated URL stops matching the browser's and bug reports become guesswork. This task is the parse half of the round-trip: the DEV and FULL URLs must both survive it field for field, and 01-04 adds the render half plus the byte-level assertion.

## Current state (preconditions)

- 01-02 is done: `models.py` has `UnitState`, `UnitCard`, `PageResult` and a `SearchSpec` with the ten canonical fields and no `extra`; `test_models.py` is green.
- `pipeline/src/connex_watch/urls.py` and `pipeline/tests/test_urls.py` do not exist. `pytest pipeline -q` is green, and the two reference URLs in `plan/01-discovery.md` §1 are the only inputs — no live fetch.

## Read before you start

- `CLAUDE.md` §2, §3, §5 (URL building lives in `urls.py`), §6.
- `plan/01-discovery.md` §4 — the query-parameter table, the canonical order, "keep empty params present", and `status` being decorative (preserve it, do not interpret it).
- `plan/02-architecture.md` §3.1 and §3.3 — `SearchSpec` is "the query params"; the published `search` block is snake_case and has no `page_no`.
- `plan/stage-01-search-parser.md` Task 1.1 — the original wording, the two URLs and the `extra`/`with_` gotcha.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/models.py` | `SearchSpec.extra` and `SearchSpec.with_(**changes)` |
| create | `pipeline/src/connex_watch/urls.py` | `parse_search_url(url) -> SearchSpec` |
| create | `pipeline/tests/test_urls.py` | the seven cases below |

Nothing else may change; `SearchSpec` keeps its snake_case keys and 01-04 owns rendering.

## Test first (red)

1. `test_from_url_parses_the_dev_url` — the DEV URL (`project_code=p00348`, `listingtype=2`, `propertytype=1`, `bedroom=`, `minsize=55`, `maxsize=`, `minprice=25000`, `maxprice=50000`, `special=`, `sort=1`, `page_no=1`) → `project_code="p00348"`, `listingtype=2`, `propertytype=1`, `bedroom=None`, `minsize=55`, `maxsize=None`, `minprice=25000`, `maxprice=50000`, `special=None`, `sort="1"`; numbers are `int`, not the raw strings.
2. `test_from_url_parses_the_full_url` — the FULL URL (empty `project_code`, `minsize=45`, `maxsize=150`, `sort=`) → `project_code=""`, `minsize=45`, `maxsize=150`, `sort=""`, `bedroom=None`, `special=None`. Empty and absent are the same thing; `sort=""` is not `None`.
3. `test_page_number_is_not_part_of_the_spec` — parsing the DEV page-1 URL and the same URL with `page_no=2` yields equal specs and no `page_no` key in `.extra`; the page is the cursor, not the search (01-04 renders it).
4. `test_unknown_params_are_kept_in_extra` — the DEV URL with `&status=1` appended → `.extra == {"status": "1"}` and the ten canonical fields unchanged, because `status` is decorative and must be preserved, not interpreted.
5. `test_spec_is_frozen` — assigning `spec.minprice = 1` raises `ValidationError`, so a crawl cannot mutate the spec it logged.
6. `test_with_returns_a_new_spec` — `spec.with_(minsize=60)` is a different object with `minsize=60` while the original still has 55 and keeps its `.extra`; one-sentence docstring on `with_`.
7. `test_from_url_rejects_a_foreign_host` — `https://example.com/search_en?...` raises `ValueError` naming the host, so a typo cannot be crawled for 546 pages.

Expected red symptom: `ModuleNotFoundError: No module named 'connex_watch.urls'` for cases 1–4 and 7; an unexpected-field `TypeError`/`ValidationError` for case 4 once the module exists. Run `. .venv/bin/activate && pytest pipeline/tests/test_urls.py -q` — seven red.

## Implement (green)

In `models.py`, keep the ten default values and add `extra: dict[str, str] = {}` (encounter order preserved) plus `with_(**changes)` returning a copy with the given fields replaced; the model stays frozen. In `urls.py`, add `parse_search_url(url: str) -> SearchSpec`:

- accept only `http`/`https` on host `connex.in.th`; anything else raises `ValueError` containing the offending host;
- read the query string only; map the ten canonical names to `int` for `listingtype`, `propertytype`, `bedroom`, `minsize`, `maxsize`, `minprice`, `maxprice`, `special` and `str` for `project_code`, `sort`; an empty value means `None` for a number and `""` for a string;
- treat `page_no` as known-but-not-stored: validate it is a positive integer, then drop it — it must never reach `.extra`;
- every other parameter goes into `.extra` verbatim, duplicates last-wins; a missing canonical parameter falls back to the model default;
- declare the canonical names, types and order in one table in `urls.py`, because 01-04 renders from the same table.

No network, no redirect resolution, no URL normalisation, no logging. Run: `. .venv/bin/activate && pytest pipeline/tests/test_urls.py -q` — seven green.

## Refactor / keep it DRY

One canonical parameter table, shared with 01-04; no second name list. `SearchSpec` must not gain a `page_no` field or a `to_url` method (that would duplicate 01-04 and invert the module dependency). Re-run the suite after the refactor.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_urls.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Manual check: parse the FULL URL and print the spec; `sort` and `project_code` must print as empty strings and `minsize` as 45, not `None`.

## Out of scope (do not do)

- `build_search_url`, canonical rendering and the byte-level round-trip — 01-04.
- Presets, `presets.toml` and CLI flags — stage 02 (02-13).
- Storing `page_no`, `fetchedAt` or the request URL on the spec — stage 02's manifest; special-casing `status=1` out of `.extra`, or dropping unknown parameters "for cleanliness".

## Definition of done

- [ ] All seven cases exist, named as written, and pass; the red run is quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only the three files above.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
01-03: parse search spec from url

- points: 1
- red: test_from_url_parses_the_dev_url failed (ModuleNotFoundError: connex_watch.urls)
- add parse_search_url with the canonical table, empty-value handling and .extra
- add SearchSpec.extra and with_(); page_no is validated and dropped, never stored
- suite: 18 passed
```

Files in the commit: `models.py`, `urls.py`, `test_urls.py`, this task file. Nothing else.

## Traps and review notes

- Do not smuggle `page_no` into `.extra` "for safety": 01-04 would render it twice and the round-trip would break in a confusing place.
- Do not turn `sort=""` into `None`; the FULL spec's default is the empty string and 01-04 renders the two differently.
- Do not parse numbers with `float` or `int(value or 0)` — an empty `bedroom=` means "any", not studio.
- Do not accept a relative or hostless URL: a crawled typo must fail here. Reviewers: grep `urls.py` for a parameter list used by exactly one function — the table must be shared-ready.

## Plan samples are untrusted

`plan/stage-01-search-parser.md` shows the chain `SearchSpec.from_url(url).to_url(page_no=1)`. This task implements `parse_search_url` as a module function in `urls.py` instead of a classmethod, because `CLAUDE.md` §5 puts URL building in `urls.py` and a `models.py` classmethod importing `urls.py` would be circular. Verify the snippet against the installed pydantic version and record the deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
