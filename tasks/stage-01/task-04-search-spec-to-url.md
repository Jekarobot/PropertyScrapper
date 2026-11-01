---
id: 01-04
stage: 01
title: "Canonical `to_url`, round-trip, unknown params"
points: 1
owner: pipeline
status: todo
depends_on: [01-03]
files:
  - pipeline/src/connex_watch/urls.py
  - pipeline/tests/test_urls.py
---

# 01-04 · Canonical `to_url`, round-trip, unknown params

**Points:** 1 · **Stage:** 01 — Search-page parser · **Owner lane:** pipeline · **Depends on:** 01-03 · **Unblocks:** none directly (02-05's page walk renders through it)

## Goal

Render a `SearchSpec` back into the site's canonical `search_en` URL — byte-identical to the URL a human copied from the browser — and prove parse → render → parse is lossless.

## Why this task exists

The crawler builds every request from the spec plus a page number, so the renderer becomes the single place a URL is constructed; no page walk may concatenate query strings by hand. A canonical order with empty parameters kept present makes generated URLs diff-friendly and byte-comparable with the ones in bug reports, and it is how pagination drift is detected: if re-rendering a page link does not reproduce the fixture's link, the site changed its scheme and we want to know immediately (risk R8). The round-trip on the two reference URLs is the strongest single guard this stage has.

## Current state (preconditions)

- 01-03 is done: `parse_search_url` exists, `SearchSpec` has `extra` and `with_()`, `test_urls.py` has seven green cases.
- No rendering function exists yet; `pytest pipeline -q` is green, and the exact expected strings are the DEV and FULL URLs in `plan/01-discovery.md` §1 — never approximate them.

## Read before you start

- `CLAUDE.md` §5 (URL building lives in `urls.py`; one implementation per rule), §6.
- `plan/01-discovery.md` §4 — the canonical order `project_code, listingtype, propertytype, bedroom, minsize, maxsize, minprice, maxprice, special, sort, page_no` and "keep empty params present".
- `plan/01-discovery.md` §3 — the pager's next link to the DEV page 2 URL, which the renderer must be able to reproduce.
- `plan/90-test-matrix.md` rows P-01 and P-02 — the requirements this task satisfies.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/urls.py` | `build_search_url(spec, page_no=1) -> str` |
| modify | `pipeline/tests/test_urls.py` | the eight cases below |

Nothing else may change; `parse_search_url` keeps its behaviour and only the ordering table may move.

## Test first (red)

1. `test_roundtrip_dev` — `build_search_url(parse_search_url(DEV_URL), page_no=1)` equals the DEV URL character for character, including `bedroom=`, `maxsize=`, `special=` and `sort=1`. Matrix row P-01.
2. `test_roundtrip_full` — the same for the FULL URL, including `project_code=`, `maxsize=150` and `sort=` with nothing after it. Matrix row P-01.
3. `test_roundtrip_second_page` — the DEV page-2 URL (identical except `page_no=2`) round-trips with `page_no=2`; fails if the renderer ignores the argument or if `parse_search_url` put `page_no` into `.extra`.
4. `test_canonical_parameter_order` — a spec with every field set renders the eleven parameters in the §4 order; assert the whole string, not a set. Matrix row P-02.
5. `test_empty_parameters_stay_present` — `bedroom=None`, `maxsize=None`, `special=None`, `project_code=""`, `sort=""` render as `bedroom=`, `maxsize=`, `special=`, `project_code=` and `sort=` — never omitted.
6. `test_page_no_is_last_and_always_set` — the default renders `page_no=1` even when the spec never mentioned a page, and `page_no=71` is the last canonical parameter; a value below 1 raises `ValueError`.
7. `test_unknown_params_are_re_emitted_after_the_canonical_block` — the DEV URL with `&status=1` renders the canonical block first and `status=1` last, and parsing that output yields a spec equal to the original.
8. `test_from_url_of_built_url_is_lossless` — for both reference specs, `parse_search_url(build_search_url(spec))` equals `spec` field for field, `.extra` included.

Expected red symptom: `ImportError: cannot import name 'build_search_url' from 'connex_watch.urls'` for every case. A one-character diff inside a rendered string is the real test failing — read the fixture before touching the assertion.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_urls.py -q` — eight red.

## Implement (green)

Add `build_search_url(spec: SearchSpec, page_no: int = 1) -> str` to `urls.py`:

- base is `https://connex.in.th/search_en` plus `?`; the path is fixed, never derived from input;
- emit the ten canonical parameters in §4 order, then `page_no`, then `.extra` in encounter order, joined with `&`;
- every canonical parameter is always emitted: `None` and `""` both render as an empty value, integers in plain decimal, so `sort=` keeps its trailing `=`;
- `page_no` is always present, defaults to 1, and must be the last canonical parameter; names and values are emitted as-is — no percent-encoding of digits or empty values, no reordering of `.extra`;
- iterate the shared canonical table from 01-03 in both directions rather than keeping a second name list; one-sentence docstring stating spec + page → URL, order, empty values kept. This function is the only URL builder in the project: stages 02, 03 and 08 call `build_search_url(spec, page_no)` directly, so keep the signature stable and do not add a `SearchSpec.to_url` method (the plan's sketch).

No logging, no network, no `urlencode` (it may drop or reorder empty values).

Run: `. .venv/bin/activate && pytest pipeline/tests/test_urls.py -q` — fifteen green.

## Refactor / keep it DRY

The canonical parameter list, its order and its value rendering exist in exactly one place used by both directions; delete any duplicated name list `parse_search_url` grew in 01-03 and re-run the whole suite.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_urls.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Manual check: print `build_search_url(SearchSpec())` and compare it with the FULL URL in `plan/01-discovery.md` §1 — identical, `sort=` and `page_no=1` included.

## Out of scope (do not do)

- Fetching, following redirects, or resolving the site's own relative links.
- Pagination walking or a `next_page` helper — stage 02 (02-05).
- URL shortening, slug normalisation, dropping default-valued parameters, percent-encoding helpers, sorting `.extra`, a `to_curl()` debug aid, or any change to `SearchSpec`.

## Definition of done

- [ ] All eight cases exist, named as written, and pass; the red run is quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `urls.py`, `test_urls.py` and this task file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
01-04: render canonical search url

- points: 1
- red: test_roundtrip_dev failed (ImportError: build_search_url)
- add build_search_url with the shared canonical order, empty values kept, page_no last
- re-emit unknown params from .extra after the canonical block
- suite: 26 passed
```

Files in the commit: `urls.py`, `test_urls.py`, this task file. Nothing else.

## Traps and review notes

- "`page_no` last" and "extra params last" both appear in the brief: resolve it as `page_no` being the last canonical parameter with unknown parameters after it, and record any other choice in `## Decisions`.
- `sort=` must render with a trailing `=` and nothing after it; a dropped parameter or `str(None)` breaks the FULL round-trip by one character.
- Do not sort `.extra` unless you also stop claiming encounter order — one deterministic rule, one aligned test. Reviewers: grep for a second hard-coded parameter list; the round-trip still passes while the two lists slowly diverge.

## Plan samples are untrusted

`plan/stage-01-search-parser.md` writes the round-trip as `from_url(url).to_url(page_no=1)`; this stage keeps URL work in `urls.py` as `parse_search_url`/`build_search_url` for the reason recorded in 01-03. Any sample that builds a query string by hand must be checked against the fixture's pager links before it is trusted; record deviations under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
