# Stage 03 — Dataset builder: clean, dedupe, group by building

**Goal:** `connex-watch build --raw data/raw/<runId>` turns raw pages into the published dataset
(without coordinates): only `available` units, one entry per building, deterministic ordering,
statistics. **No network.**

**Depends on:** Stage 01 (models). Can run in parallel with Stage 02 — the builder reads raw JSON
from disk, and Stage 02's tests already produce a fixture run directory.

**Files:**

```text
pipeline/src/connex_watch/build_dataset.py
pipeline/src/connex_watch/normalize.py      (name/slug/district helpers)
pipeline/src/connex_watch/validate.py       (dataset invariants = the `check` command)
pipeline/tests/test_normalize.py
pipeline/tests/test_build_dedupe.py
pipeline/tests/test_build_group.py
pipeline/tests/test_build_dataset.py
pipeline/tests/test_validate.py
pipeline/tests/fixtures/run-mini/           (hand-built raw run: 3 pages, 9 cards, 4 buildings)
```

The `run-mini` fixture is committed as raw JSON (small, hand-written, derived from the real
fixtures) so this stage needs no crawler:

| Page | Cards | Purpose |
|---|---|---|
| 1 | 4 available: `CX-1` (A Tower, 45k, 60 sqm, 1BR), `CX-2` (A Tower, 50k, 65 sqm, 1BR), `CX-3` (B Place, 32k, 55 sqm, 2BR), `CX-4` (`a tower` lowercase, 48k, 58 sqm, 1BR) | case-variant merging |
| 2 | 2 available: `CX-5` (Condo, A, B , Khlong Toei , Bangkok), `CX-1` again (duplicate id, price 44k) | duplicate id, comma in name |
| 3 | 3 non-available: `CX-6` check, `CX-7` not, `CX-8` not | exclusion + counts |

---

## Task 3.1 — Normalisation helpers

**Contract**

```python
def normalize_name(raw: str) -> str        # NFKC, collapse whitespace, casefold, strip punctuation at ends
def slugify(name: str) -> str              # "The Bangkok Sathorn" -> "the-bangkok-sathorn"
def display_name(variants: Iterable[str]) -> str   # most frequent original, tie -> lexicographic
def parse_location(parts: list[str]) -> tuple[str, str | None, str]   # building, district, province
```

**TDD order** (`test_normalize.py`):

1. `normalize_name("  The  Bangkok   Sathorn ") == "the bangkok sathorn"`.
2. Unicode: `normalize_name("เดอะบางกอกสาทร")` is stable under NFC/NFD input (apply NFKC, compare).
3. Punctuation: `"The Bangkok Sathorn."` == `"the bangkok sathorn"` (trailing dot dropped).
4. `slugify` → lowercase, `-` separators, no leading/trailing `-`, Thai characters preserved
   (not transliterated), collisions handled by the caller (Stage 04), max length 80.
5. `display_name(["The Bangkok Sathorn", "the bangkok sathorn", "The Bangkok Sathorn"])` →
   `"The Bangkok Sathorn"`; a 1-1 tie → lexicographically smallest.
6. `parse_location(["A, B Condo", "Khlong Toei", "Bangkok"])` → `("A, B Condo", "Khlong Toei", "Bangkok")`;
   `["X", "Bangkok"]` → `("X", None, "Bangkok")`; `["X"]` → `("X", None, None)`; empty → `("", None, None)`.

Commit: `stage-03: name normalisation helpers`.

---

## Task 3.2 — Filter and dedupe

**TDD order** (`test_build_dedupe.py`, input `run-mini`):

1. Only `available` units survive; `excluded == {"check_availability": 1, "not_available": 2}`.
2. `CX-1` appears on pages 1 and 2 → exactly one unit remains, the **first** occurrence wins
   (price 45 000, not 44 000), `duplicates_dropped == 1`.
3. A card missing `size_sqm` or `price_thb` is dropped and counted in
   `dropped_invalid` (log a warning with the id) — never crash the whole build.
4. Zero available units → the builder raises `EmptyDatasetError` (guards against publishing an empty
   file when the site layout breaks).
5. Deterministic: running twice on the same input yields identical JSON except `generatedAt`.

Commit: `stage-03: filter available units and dedupe`.

---

## Task 3.3 — Group by building

**TDD order** (`test_build_group.py`):

1. `CX-1`, `CX-2`, `CX-4` (case variant) → **one** building `A Tower` with 3 units, `priceMin 45 000`,
   `priceMax 50 000`, `sizeMin 58`, `sizeMax 65`.
2. `CX-3` → `B Place`, 1 unit.
3. Building `id` before enrichment = `slugify(display_name)`; stable across runs; unique (collision
   gets `-2` suffix and a warning).
4. `district`/`province` per building = most frequent non-null value among its units; if all null →
   `null`.
5. Ordering: buildings by `name` case-insensitive then `id`; units by `priceThb` then `id`.
6. `unitCount`, `priceMin/Max`, `sizeMin/Max` are recomputed from units, never trusted from a card.

Commit: `stage-03: group units into buildings`.

---

## Task 3.4 — Assemble the dataset and the `check` command

**Files:** `build_dataset.py::build_dataset(run_dir) -> Dataset`, `validate.py::validate_dataset(ds) -> list[str]`,
CLI `connex-watch build --raw DIR --out FILE [--no-coords]` and `connex-watch check --dataset FILE`.

**TDD order** (`test_build_dataset.py`, `test_validate.py`):

1. `build_dataset(run-mini)` returns a `Dataset` whose `schemaVersion == 1`, `stats.units == 5`,
   `stats.buildings == 3`, `stats.duplicatesDropped == 1`, `stats.pagesScanned == 3`,
   `stats.partial == False`, `search` mirrors the manifest spec.
2. JSON dump → `json.loads` → equals the model dump (no `datetime` leaking as non-ISO).
3. `validate_dataset` returns `[]` for a good dataset and these violations for bad ones:
   duplicate unit id; unit url not starting with `https://connex.in.th/`; `priceThb <= 0`;
   `sizeSqm <= 0`; `unitCount != len(units)`; `priceMin > priceMax`; `generatedAt` naive datetime;
   `stats.units != sum(unitCount)`.
4. CLI `build` writes the file atomically and prints `units=… buildings=… pages=… partial=…`.
5. CLI `check` exits 0/1 with one line per violation (rule P9 in `02-architecture.md` §6).

Commit: `stage-03: assemble dataset and validation command`.

---

## Task 3.5 — Edge cases round

Add tests (and only the fixes they demand) for:

- a building whose every unit is `check_availability` → not in the dataset at all;
- `priceThb` of exactly the filter bounds (25 000 / 50 000) → kept (inclusive);
- two buildings that differ only by a trailing `" Tower"` → **not** merged (no fuzzy matching; YAGNI);
- a unit whose URL has a query string → stored verbatim (we never rebuild URLs);
- 0-byte page files in the raw dir → skipped with a warning, `stats.partial = True`.

Commit: `stage-03: edge cases for grouping`.

---

## Acceptance criteria

- [ ] `pytest pipeline -q` green; total ≥ 70
- [ ] `connex-watch build --raw pipeline/tests/fixtures/run-mini --out /tmp/ds.json` → valid JSON, `check` exits 0
- [ ] dataset contains no `check_availability`/`not_available` unit
- [ ] building count for `run-mini` == 3 (`A Tower`, `B Place`, `A, B Condo`)
- [ ] `build_dataset.py` imports neither `httpx` nor `fetch`
- [ ] output is byte-identical across two runs when `generatedAt` is injected (test with a fixed clock)

## Result

| Task | Tests | Status |
|---|---|---|
| 3.1 | `test_normalize.py` (7) | |
| 3.2 | `test_build_dedupe.py` (5) | |
| 3.3 | `test_build_group.py` (6) | |
| 3.4 | `test_build_dataset.py` + `test_validate.py` (10) | |
| 3.5 | `test_build_edge.py` (5) | |

## Traps

- Do not merge case-variants by fuzzy matching — normalise the key, keep the display name as the
  most common original string.
- `stats.excluded` counts units seen on pages we actually scanned, so it is a **lower bound** for the
  whole site; name it accordingly in the UI ("seen, excluded") and keep it out of user-facing copy.
- Determinism: sort before dumping, and inject `now()` so tests can freeze time.
- The dataset has no coordinates yet; `lat`/`lng` fields must already exist as `null` so Stage 04 is
  purely additive and Stage 05 can start against the sample dataset.
