---
id: 01-12
stage: 01
title: Unit id, absolute URL, image URL
points: 1
owner: pipeline
status: todo
depends_on: [01-05]
files:
  - pipeline/src/connex_watch/search_parser.py
  - pipeline/tests/test_search_parser_identity.py
---

# 01-12 · Unit id, absolute URL, image URL

**Points:** 1 · **Stage:** 01 — Search-page parser · **Owner lane:** pipeline · **Depends on:** 01-05 · **Unblocks:** 01-13, 01-14

## Goal

Give every card a normalised unit id, its detail URL exactly as the site printed it, and its absolute cover-image URL.

## Why this task exists

The unit id is the dedupe key of the whole pipeline: the crawler counts duplicates by it, the dataset sorts by it, and a bug report is identified by it. The site prints it in two shapes — lowercase in the detail URL slug, uppercase with a prefix in the image path and the image `alt` — so exactly one normalisation has to exist or two cards for the same flat will look like two flats. The detail URL must be stored verbatim because the slug embeds SEO words and the URL scheme may change (risk R8): rebuilding it from parts would break every unit link at once.

## Current state (preconditions)

- 01-05 is done: handles carry the anchor `href` verbatim, 1-based positions and the card node.
- `UnitCard.id`, `url` and `image_url` are required strings (01-02); nothing fills them yet.
- `pipeline/tests/test_search_parser_identity.py` does not exist; `pytest pipeline -q` is green.
- Fixture facts: card #1 is `CX-84121` with image `…/uploads/listing/CX-84121/webp/mini_w500/01_h20zs.webp`; card #9 is `CX-00427` with image `…/CX-00427/webp/mini_w500/1.webp`.

## Read before you start

- `CLAUDE.md` §3, §5, §6.
- `plan/01-discovery.md` §2 — the unit-link, unit-id and image rows of the selector table, plus the note that the id is normalised to `CX-84121`.
- `plan/01-discovery.md` §9 — the URL slug contains SEO words and must not be parsed for the building name; §10 risk R8 (store the absolute URL, never rebuild it).
- `plan/02-architecture.md` §3.1–§3.2 — `UnitCard` identity is `CX-<digits>`; `id`, `url`, `imageUrl` in the raw page JSON.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| modify | `pipeline/src/connex_watch/search_parser.py` | `parse_unit_id`, `parse_unit_url`, `parse_image_url` |
| create | `pipeline/tests/test_search_parser_identity.py` | the six cases below |

Nothing else may change: no location, no date, no card assembly.

## Test first (red)

1. `test_unit_id_of_the_first_card_is_cx_84121` — `search_dev_p1` position 1 → `CX-84121`, uppercase prefix, digits unchanged.
2. `test_unit_id_keeps_leading_zeros` — position 9 → `CX-00427`; converting the digits to an integer or re-padding them would produce `CX-427` and break dedupe against the image path.
3. `test_unit_url_is_the_anchor_href_unchanged` — position 1 → exactly `https://connex.in.th/en/detail/for-rent-condo-the-bangkok-sathorn-high-floor-river-view-bts-surasak-yannawa-sa-thon-bangkok-cx-84121`, with no rebuilt path, no trailing slash and no lowercase rewriting.
4. `test_image_url_is_absolute_and_unchanged` — position 1 → exactly `https://connex.in.th/uploads/listing/CX-84121/webp/mini_w500/01_h20zs.webp`.
5. `test_image_url_is_not_synthesised_from_the_id` — position 9 → `https://connex.in.th/uploads/listing/CX-00427/webp/mini_w500/1.webp`; the file name is `1.webp`, not a zero-padded copy of the id, so a template built from the id fails here.
6. `test_ids_are_unique_on_a_page` — the 16 ids of `search_dev_p1` are all distinct, which is the property the dedupe relies on.

Expected red symptom: `ImportError: cannot import name 'parse_unit_id' from 'connex_watch.search_parser'` for cases 1, 2 and 6, and the same for `parse_unit_url`/`parse_image_url` in 3–5.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_identity.py -q` — six red.

## Implement (green)

In `search_parser.py`:

- `parse_unit_id(card_node: CardNode) -> str` — take the digits from the end of the handle's `href` (the slug ends in `-cx-<digits>`) and return the normalised `CX-<digits>`: uppercase `CX`, hyphen, the digits exactly as printed, including leading zeros. When the href does not match, raise `ParseError("identity", position)` rather than falling back to the image `alt`.
- `parse_unit_url(card_node: CardNode) -> str` — return the handle's `href` unchanged (whitespace already trimmed by 01-05); no normalisation, no rebuilding.
- `parse_image_url(card_node: CardNode) -> str` — return the `coverimg_2` `src` attribute as printed; the site prints absolute URLs, so the function asserts absolute-ness and raises `ParseError("image", position)` on a relative `src` instead of prefixing a host.
- one sentence of docstring per function stating the source, the normalisation and the error.

The id pattern exists once, here — no second copy in the tests' helpers or in a model validator.

Run: `. .venv/bin/activate && pytest pipeline/tests/test_search_parser_identity.py -q` — six green.

## Refactor / keep it DRY

The id regex lives only in `parse_unit_id`; the URL and image functions must not re-derive anything from it. No helper that rebuilds a detail URL from an id and a slug — that is the R8 trap. Re-run the suite after the refactor.

## Verification (how to test it)

```text
. .venv/bin/activate && pytest pipeline/tests/test_search_parser_identity.py -q
. .venv/bin/activate && pytest pipeline -q
ruff check pipeline
```

Manual check: print the ids and image URLs of all 16 cards; every id is `CX-<digits>` and every image URL starts with `https://connex.in.th/uploads/listing/` and contains the same digits as its id.

## Out of scope (do not do)

- Parsing the building name or the location out of the slug — 01-13 uses the location line.
- Fetching the detail page, following the URL, checking it is still live, rewriting the URL to a canonical form, stripping query strings or resolving `http`.
- A date parser (01-14), an image download, a thumbnail size variant, or an `image_filename` field.

## Definition of done

- [ ] All six cases exist, named as written, and pass; the red run is quoted in the commit body.
- [ ] `pytest pipeline -q` green; `ruff check pipeline` clean.
- [ ] `git status --short` lists only `search_parser.py`, `test_search_parser_identity.py`, this file.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and test counts.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
01-12: parse unit id and absolute urls

- points: 1
- red: test_unit_id_of_the_first_card_is_cx_84121 failed (ImportError: parse_unit_id)
- add parse_unit_id from the slug tail, keeping leading zeros
- add parse_unit_url and parse_image_url as verbatim absolute strings
- suite: 71 passed
```

Files in the commit: `search_parser.py`, `test_search_parser_identity.py`, this task file. Nothing else.

## Traps and review notes

- Do not read the id from the image `alt` or the image path: the slug tail is the canonical source and the other two diverge first.
- Do not `int()` the digits: `00427` would become `427` and stop matching the image path.
- Do not build the detail URL from a slug template, and do not `urljoin` it against a hard-coded host.
- Do not make the id pattern a `UnitCard` validator now — normalisation belongs to the parser. Reviewers: grep for a second `cx-` pattern and for any `connex.in.th` concatenation outside `urls.py`.

## Plan samples are untrusted

`plan/stage-01-search-parser.md` Task 1.4 lists the expected id, URL and image values but shows no code; the fixture is the contract. Verify any regular expression against the installed Python `re` flavour and the fixture's lower-case slug, and record any deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |
