# Stage 02 — Crawler: fetch, throttle, stop, checkpoint

**Goal:** `connex-watch crawl --preset daily` walks the search result pages politely, saves every
page as JSON, stops at the right moment, and resumes after a crash. Emits **raw** data only — no
grouping, no filtering decisions beyond recording states.

**Depends on:** Stage 01. **Est:** 60 min. **Agent:** A. **Files:**

```text
pipeline/src/connex_watch/fetch.py
pipeline/src/connex_watch/crawl.py
pipeline/src/connex_watch/store.py
pipeline/src/connex_watch/cli.py
pipeline/presets.toml
pipeline/tests/test_fetch.py
pipeline/tests/test_crawl_stop.py
pipeline/tests/test_store_resume.py
pipeline/tests/test_cli_crawl.py
pipeline/tests/test_live_smoke.py        (marked network, skipped by default)
```

---

## Task 2.1 — `ThrottledFetcher` (the only door to the network)

**Contract**

```python
class Fetcher(Protocol):
    def get(self, url: str) -> str: ...

class ThrottledFetcher:
    def __init__(self, *, min_interval=1.2, jitter=0.3, timeout=20.0, max_retries=3,
                 user_agent="ConnexWatch/1.0 (+internal tool)", sleep=time.sleep,
                 clock=time.monotonic, transport=None) -> None: ...
```

**TDD order** (`test_fetch.py`, using `httpx.MockTransport` and injected `sleep`/`clock` — no real
time, no real sockets):

1. Throttling: the **first** `get()` does not sleep; the **second** sleeps a value in
   `[min_interval, min_interval+jitter]` (the fake `sleep` records calls, so the assertion is exact).
2. `min_interval` floor: constructing with `min_interval=0.2` raises `ValueError` (politeness is not
   configurable below 1.0 s).
3. `503` → retried with exponential backoff (`2^n`), max 3 attempts, then `FetchError` carrying the
   status and URL.
4. `429` with `Retry-After: 5` → sleeps `5` (fake sleep records it) before the retry.
5. `403` → **no retry**, immediate `BlockedError` (stop hammering a site that says no).
6. `404` → `FetchError`, no retry.
7. Response decoding is forced to UTF-8 and non-UTF8 bytes do not crash (the site serves UTF-8).
8. `get()` returns `str`, not bytes; no cookies are required (assert the client is constructed with
   `follow_redirects=True` and a 20 s timeout).

**Then implement.** `fetch.py` must not import `search_parser` (single responsibility) and must be
the only module importing `httpx` — the reviewer greps for that.

Commit: `stage-02: throttled fetcher with retries`.

---

## Task 2.2 — Stop policy and page walk

**Contract**

```python
@dataclass(frozen=True)
class StopPolicy:
    max_pages: int | None = None
    empty_streak: int = 2          # pages with 0 available-after-filter cards
    scan_all: bool = False         # --full

class Crawler:                     # created by crawl()
    def run(self, spec: SearchSpec) -> CrawlResult: ...
```

**TDD order** (`test_crawl_stop.py`, `FakeFetcher` that maps `page_no` → fixture text and records
calls):

| # | Scenario | Expected |
|---|---|---|
| 1 | DEV spec: page 1 (4 available), page 2 (0 available), then out-of-range stub | fetches pages **1,2,3** only; `stop_reason == "empty_streak"`; 3 requests; 32 cards parsed |
| 2 | FULL spec: p1 (16 avail) → p71 (12 avail) → p72 (0) → p73 (0) → stop | stops after p73; `stop_reason == "empty_streak"` |
| 3 | `empty_streak=1` | stops after the first 0-available page (one page earlier) |
| 4 | page 1 has `total_pages == 1` | exactly 1 request, `stop_reason == "last_page"` |
| 5 | `max_pages=2` | 2 requests, `stop_reason == "max_pages"` |
| 6 | `scan_all=True` | walks to `total_pages`, ignoring streaks (fake has 3 pages) |
| 7 | page 3 returns the `error/th` stub while `total_pages` claims 5 | treated as end-of-data, `stop_reason == "error_stub"`, `result.partial is True` |
| 8 | page 2 returns **0 cards but is not beyond `total_pages`** | raises `ParseError` — this is R1 (site changed), not end-of-data |
| 9 | the same page is fetched twice in one run (faulty pager) | never happens: the walk is driven by our own counter; assert call sequence is strictly 1..n |
| 10 | duplicate unit ids across pages | allowed at this layer; only counted (`CrawlResult.duplicate_units`) |

**Then implement:** a `while` loop over `page_no`, parsing each page, accumulating
`CrawlResult(cards, pages, state_histogram, stop_reason, request_count, partial)`.

**Non-negotiables:** sequential requests only; no threads, no asyncio; a 0-card page that is not
beyond `total_pages` is an error; the loop always honours `max_pages`.

Commit: `stage-02: crawl loop with stop policy`.

---

## Task 2.3 — Checkpoint store and resume

**Files:** `store.py`. Layout from `02-architecture.md` §3.2.

**TDD order** (`test_store_resume.py`, `tmp_path`):

1. `RunStore.create(run_id, spec)` creates `data/raw/<runId>/pages/` and writes `manifest.json` with
   `startedAt`, `spec`, `stopReason=None`.
2. `save_page(result)` writes `pages/page-0001.json` atomically: after the call no `*.tmp` file
   remains; JSON validates against the raw schema (round-trips through `PageResult`).
3. `save_page` twice for the same page overwrites, never duplicates.
4. `has_page(n)` / `load_page(n)` work for written pages; `load_page` of a missing page → `None`.
5. `finish(stop_reason, partial)` sets `finishedAt`, `pagesFetched`, `stateHistogram`, `requestCount`.
6. Resume: `crawl(..., resume=True)` with pages 1–2 already on disk fetches **only** page 3, and the
   final `CrawlResult` contains the cards from all three pages.
7. Resume with a manifest whose `spec` differs from the requested spec → `ValueError` (never mix two
   searches in one run directory).

Commit: `stage-02: checkpoint store and resume`.

---

## Task 2.4 — CLI `crawl`

**Files:** `cli.py`, `presets.toml`.

```toml
[presets.dev]
url = "https://connex.in.th/search_en?project_code=p00348&listingtype=2&propertytype=1&bedroom=&minsize=55&maxsize=&minprice=25000&maxprice=50000&special=&sort=1&page_no=1"

[presets.daily]
url = "https://connex.in.th/search_en?project_code=&listingtype=2&propertytype=1&bedroom=&minsize=45&maxsize=150&minprice=25000&maxprice=50000&special=&sort=&page_no=1"
```

**Contract**

```text
connex-watch crawl --preset daily|dev | --url <search url>
                   [--out data/raw] [--run-id ID] [--max-pages N] [--full]
                   [--delay 1.2] [--resume] [--dry-run] [-v]
```

**TDD order** (`test_cli_crawl.py`, subprocess-free: call `main(argv)` and assert `SystemExit.code`
plus files on disk, with the fetcher injected via a factory hook `cli._make_fetcher`):

1. `--dry-run` prints the URLs for pages 1..N (from the pager) and makes **no** requests
   (`FakeFetcher.calls == []`).
2. `--preset nope` → exit code 2 with a message listing available presets.
3. successful crawl → exit 0, prints one summary line: pages, cards, per-state counts, stop reason,
   duplicates, elapsed.
4. `BlockedError` → exit code 3, message tells the operator to stop and re-run later.
5. `ParseError` → exit code 4, message names the page and the failing field.
6. `--delay 0.2` → clamped to 1.0 and a warning is printed (politeness floor).
7. logging: with `-v`, one line per page `page=7/546 avail=16 check=0 not=0 req=7`.

**Exit-code table goes into `README.md`** (Stage 08 links it).

Commit: `stage-02: crawl cli with presets`.

---

## Task 2.5 — Opt-in live smoke test

`test_live_smoke.py`, marked `@pytest.mark.network`, skipped unless `-m network`:

- fetch DEV page 1 with the real fetcher, parse it, assert `>= 1` card and `page_no == 1`;
- assert the state histogram contains all three keys;
- runtime < 30 s.

This is the **only** test allowed to touch the network. `make check` runs `-m "not network"`.

Commit: `stage-02: live smoke test (opt-in)`.

---

## Acceptance criteria

- [ ] `pytest pipeline -q` green offline; total ≥ 45 tests
- [ ] `pytest pipeline -m network -q` green when run manually (1 test)
- [ ] `connex-watch crawl --preset dev --dry-run` lists exactly 2 URLs
- [ ] a real `connex-watch crawl --preset dev` (manual, owner-approved) produces
      `data/raw/<id>/{manifest.json,pages/page-0001.json,pages/page-0002.json}` and 32 cards
- [ ] `grep -rn "httpx" pipeline/src | grep -v fetch.py` → empty
- [ ] no `time.sleep` outside `fetch.py` (injectable there)
- [ ] `ruff check pipeline` clean

## Result

| Task | Tests | Status |
|---|---|---|
| 2.1 | `test_fetch.py` (8) | |
| 2.2 | `test_crawl_stop.py` (10) | |
| 2.3 | `test_store_resume.py` (7) | |
| 2.4 | `test_cli_crawl.py` (7) | |
| 2.5 | `test_live_smoke.py` (1, network) | |

## Traps

- Do not derive the next page from the pager's "next" link — build it from the spec. The link is
  sometimes absent on the last page and occasionally malformed.
- `empty_streak` counts pages **without available cards**, not pages without cards. A page of
  `check_availability` units must increment the streak.
- A page with cards but `total_pages` smaller than the requested `page_no` means the pager lied;
  trust the card count and log a warning.
- Never write partial JSON: `json.dumps` → `.tmp` → `os.replace`.
- `--resume` must still re-read the manifest to know `total_pages`, because the pager drives
  `scan_all` and `stop_reason`.
