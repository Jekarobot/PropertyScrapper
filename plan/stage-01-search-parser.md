# Stage 01 — Search-page parser (pure, offline)

**Goal:** turn a saved search page into typed `PageResult`/`UnitCard` objects, with 100 % of the
quirks from `01-discovery.md` covered by tests. **No network, no pagination, no storage.**

**Why this stage is pure:** it is the highest-risk code (site markup). Making it a pure function of
HTML lets every future change be diagnosed by one failing test, offline, in milliseconds.

**Depends on:** Stage 00. **Est:** 60 min. **Agent:** A.

**Files to create**

```text
pipeline/src/connex_watch/models.py
pipeline/src/connex_watch/urls.py
pipeline/src/connex_watch/search_parser.py
pipeline/tests/test_models.py
pipeline/tests/test_urls.py
pipeline/tests/test_search_parser_states.py
pipeline/tests/test_search_parser_fields.py
pipeline/tests/test_search_parser_identity.py
pipeline/tests/test_search_parser_pagination.py
pipeline/tests/test_search_parser_golden.py
```

---

## Task 1.1 — Models and URL builder

**TDD order**

1. `test_models.py`
   - `UnitState` has exactly three members with string values `available`, `check_availability`,
     `not_available`; `UnitState.from_label("Not available") == NOT_AVAILABLE`;
     `from_label("Check Availability")`; `from_label("Available")`; unknown label raises
     `ValueError` with the label in the message.
   - `UnitCard` rejects `price_thb <= 0`; accepts `bathrooms=None`, `floor=None`, `district=None`.
   - `SearchSpec.defaults()` yields the FULL query values (`listingtype=2`, `propertytype=1`,
     `minsize=45`, `maxsize=150`, `minprice=25000`, `maxprice=50000`, `sort=""`).
2. `test_urls.py`
   - `SearchSpec.from_url(SEARCH_URL).to_url(page_no=1) == SEARCH_URL` for **both** the DEV and FULL
     URLs (round-trip is the strongest single test we have).
   - Parameter order matches `01-discovery.md` §4 exactly, empty params stay present (`&bedroom=`).
   - `page_no` is always last and always set; `sort=""` renders as `sort=`.
   - Unknown query params are preserved in `.extra` and re-emitted last (forward compatible).
   - `spec.with_(minsize=60)` returns a new instance (frozen dataclass/pydantic `model_copy`).

**Then implement** `models.py` / `urls.py`. Run: `pytest pipeline/tests/test_models.py pipeline/tests/test_urls.py -q`.

**Gotchas:** the site tolerates missing params, but our tests pin the canonical form, because
`from_url(to_url(x))` must be lossless — that is also how we detect pagination drift later.

Commit: `stage-01: models and search url builder`.

---

## Task 1.2 — Tri-state availability and card discovery

**TDD order** (`test_search_parser_states.py`)

1. `parse_search_page(search_dev_p1) .cards` → `len == 16`.
2. State histogram of that page → `{available: 4, check_availability: 5, not_available: 7}`.
3. `search_dev_p2` → `{available: 0, check_availability: 1, not_available: 15}`.
4. `search_boundary_p71` → `{available: 12, check_availability: 4, not_available: 0}`.
5. A card whose state `<div>` was removed (synthetic: take the first card's HTML, delete the
   `status-listing` block) must **not** be silently mislabelled: the parser raises
   `ParseError("state")` mentioning the card position, **or** returns the card with state `None` —
   pick one and pin it in the test. Recommended: raise, because a silent `None` state becomes a
   silent "not available" downstream.
6. `parse_search_page(search_error_page).cards == []` and `.is_error_stub is True`.

**Then implement** `parse_search_page(html: str) -> PageResult` with
`_parse_card(node) -> UnitCard`. Every card must carry `position` (1-based) so errors are locatable.

Commit: `stage-01: parse card states`.

---

## Task 1.3 — Card fields: money, rooms, floor, size

**TDD order** (`test_search_parser_fields.py`), using card #1 of `search_dev_p1`:

| Assertion | Source |
|---|---|
| `price_thb == 45000` | `฿45,000 <span></span>` |
| `bedrooms == 1`, `bathrooms == 1`, `floor == 27`, `size_sqm == 65` | `option` table |
| a card whose size cell is `" <span>64</span> sqm."` → `64` | `search_boundary_p71` last card |
| price parsing ignores `<span>Condo</span>` and any thousands separator | e.g. `฿120,000` |
| `size_sqm` accepts decimals (`55.5`) if present | synthetic |
| missing `Floors` cell → `floor is None` (not 0) | synthetic |
| missing size → `ParseError`, because size is a filter dimension and a null would corrupt the UI |

**Implementation notes (do not over-engineer):**

- Price: first `\d[\d,]*` in `div.detail_1` text, remove commas.
- Rooms/floor: locate by the icon's `alt` (`Bedroom`, `Bathroom`, `Floors`) → nearest `td` → first
  integer in its `p` text.
- Size: the `td` whose text ends with `sqm.` (the icon `alt` is empty — see `01-discovery.md` §2).

Commit: `stage-01: parse price, rooms, floor, size`.

---

## Task 1.4 — Identity, location line, dates

**TDD order**

| Assertion | Expectation |
|---|---|
| `id` | `"CX-84121"` (uppercase `CX-`, digits preserved, no zero padding changes) |
| `url` | absolute `https://connex.in.th/en/detail/…-cx-84121`, exactly as in the anchor `href` |
| `building_name` | `"The Bangkok Sathorn"` |
| `district` / `province` | `"Sa Thon"` / `"Bangkok"` |
| `updated_on` | `date(2026, 10, 29)` — `29/10/2569` minus 543 years |
| `image_url` | absolute `…/uploads/listing/CX-84121/webp/mini_w500/01_h20zs.webp` |
| location with 2 parts (`"X , Bangkok"`) | `district is None`, `province == "Bangkok"` |
| location with a comma inside the building name (`"A, B Condo , Khlong Toei , Bangkok"`) | building `"A, B Condo"`, district `Khlong Toei` — parse from the **right** |
| date `29/10/2569` and `01/01/2500` | hold; malformed date (`"Updated n/a"`) → `updated_on is None`, no exception |

Commit: `stage-01: parse identity, location, dates`.

---

## Task 1.5 — Pagination and end-of-data

**TDD order** (`test_search_parser_pagination.py`)

1. `parse_search_page(search_dev_p1)` → `page_no == 1`, `total_pages == 2`.
2. `search_boundary_p71` → `71`, `546`.
3. `search_error_page` → `is_error_stub is True`, `page_no` unknown/None, `cards == []`.
4. A page with `.number-page` missing entirely → `ParseError("pagination")` (protects R1).
5. `total_units` is **derived** from `total_pages` and card counts (`16·(n−1)+k` as a hint only);
   never from the SEO `<meta description>`. Assert the parser does not read `meta`.

Commit: `stage-01: parse pagination and error stub`.

---

## Task 1.6 — Golden snapshot

**Files:** `pipeline/tests/test_search_parser_golden.py`,
`pipeline/tests/snapshots/search_dev_p1.json`.

1. Parse `search_dev_p1` and compare `[c.model_dump(mode="json") for c in cards]` with the snapshot
   file (committed JSON).
2. Snapshot is generated once by an explicit command (`pytest --snapshot-update` via a small
   `--write-snapshot` flag or an env var `UPDATE_SNAPSHOTS=1`), **never** by the test itself on
   failure.
3. The test message must say: `parser output changed; if the site changed, re-capture the fixture
   and review the diff; if not, fix the parser`.

This snapshot is the regression net for Stages 02–04 and the acceptance evidence for the stage.

Commit: `stage-01: add golden snapshot for dev search page`.

---

## Acceptance criteria

- [ ] `pytest pipeline -q` → **≥ 25 tests**, all green, run in < 2 s, offline
- [ ] `parse_search_page` is a pure function: no I/O, no globals, no clock (`datetime.now` absent)
- [ ] every field in `01-discovery.md` §2 selector table appears in a test
- [ ] `ruff check pipeline` clean
- [ ] `grep -rn "requests\|httpx" pipeline/src/connex_watch/search_parser.py` → empty
- [ ] golden snapshot committed and referenced in the stage Result table

## Result

| Task | Tests | Status |
|---|---|---|
| 1.1 | `test_models.py`, `test_urls.py` (8) | |
| 1.2 | `test_search_parser_states.py` (6) | |
| 1.3 | `test_search_parser_fields.py` (7) | |
| 1.4 | `test_search_parser_identity.py` (8) | |
| 1.5 | `test_search_parser_pagination.py` (5) | |
| 1.6 | `test_search_parser_golden.py` (1) | |

## Traps

- Do not split the HTML on the `col-xl-3` wrapper (the discovery doc did that only for probing
  convenience); select `.condo-card` and walk up to the `a[href]`.
- `lxml` will happily recover broken markup — that is why "0 cards" must be an explicit error path,
  not an empty list.
- The size icon's `alt` is empty: matching by `alt` will silently select the wrong cell.
- Never `float()` prices; keep integer THB.
- Do not normalise or title-case the building name here — that is Stage 03's job, and the snapshot
  must show exactly what the site says.
