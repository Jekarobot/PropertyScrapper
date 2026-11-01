# 01 — The connex.in.th contract (what the parsers must rely on)

All measurements were taken **2026-11-01** from this machine with a desktop Chrome UA. Numbers
drift; **structure and quirks are what matter**. Fixtures in `assets/fixtures/` freeze a
representative snapshot so tests never depend on the live site.

Reference URLs used throughout:

```text
DEV   (small, 2 pages)   https://connex.in.th/search_en?project_code=p00348&listingtype=2&propertytype=1&bedroom=&minsize=55&maxsize=&minprice=25000&maxprice=50000&special=&sort=1&page_no=1
FULL  (big, 546 pages)   https://connex.in.th/search_en?project_code=&listingtype=2&propertytype=1&bedroom=&minsize=45&maxsize=150&minprice=25000&maxprice=50000&special=&sort=&page_no=1
DETAIL                   https://connex.in.th/en/detail/for-rent-condo-the-bangkok-sathorn-high-floor-river-view-bts-surasak-yannawa-sa-thon-bangkok-cx-84121
```

---

## 1. There is no JSON API — confirmed three ways

| Check | Result |
|---|---|
| `grep 'fetch(' / '$.ajax' / '$.post'` on a search page | 0 hits |
| Same on a detail page | 0 hits; the only `.php` URLs are `include/process_contact.php` (a contact form) and share links |
| `POST /search_en` with the filter form body | `200` but a 74-byte stub: `<script>window.location.replace('https://connex.in.th/error/th')</script>`. `POST` to the URL **including** its query string returns a stub that redirects back to the same GET URL. The site is GET-only. |

**Consequence:** the only data channel is the HTML of `search_en` pages (~17 KB gzip each) plus the
HTML of unit detail pages (~24 KB gzip each, needed only for coordinates).

---

## 2. Anatomy of a search page

One page = 16 result cards, each wrapped in its own `<a>`:

```html
<div class="col-xl-3 col-lg-6 col-md-6 col-sm-6 col-12">
  <a href="https://connex.in.th/en/detail/…-cx-84121">
    <div class="condo-card">
      <div class="cover">
        <div class="status-listing">
          <div class="label-status-listing">
            <div class="available"><span>Available</span></div>   <!-- or not-available / check-availability -->
          </div>
        </div>
        <div class="status-update"><span>Updated 29/10/2569</span></div>
        <img class="coverimg_2" src="…/uploads/listing/CX-84121/webp/mini_w500/01_h20zs.webp">
      </div>
      <div class="detail">
        <div class="detail_1">฿45,000 <span></span></div>
        <h2><span>Rent</span><div class="title_h2">The Bangkok Sathorn</div></h2>
        <div class="detail_3">The Bangkok Sathorn , Sa Thon , Bangkok</div>
      </div>
      <div class="option">
        <table><tr>
          <td><img alt="Bedroom"><p>Bedroom <span>1 </span></p></td>
          <td><img alt="Bathroom"><p>Bathroom <span>1 </span></p></td>
          <td><img alt="Floors"><p>Floors <span>27</span></p></td>
          <td><img alt=""><p>  <span>65</span> sqm.</p></td>       <!-- NOTE: size icon has EMPTY alt -->
        </tr></table>
      </div>
    </div>
  </a>
</div>
```

### Selector table (use exactly these; prefer the narrowest one)

| Field | Selector | Notes |
|---|---|---|
| card roots | `.condo-card` | 16 per page; do **not** split on the `col-xl-3` wrapper in production code |
| unit link | card's parent `a[href]` | absolute: `https://connex.in.th/en/detail/<slug>-cx-<digits>` |
| unit id | `cx-(\d+)$` on the link, or `CX-\d+` in `img.coverimg_2[alt]` | normalise to `CX-84121` |
| state | `.status-listing .available` / `.not-available` / `.check-availability` | exactly one per card; 0 cards in 96 sampled lacked one |
| state label | the `<span>` inside the state div | `Available` / `Not available` / `Check Availability` |
| price | `.detail_1` text | `฿45,000`; may be followed by `<span>Condo</span>`; parse digits only |
| building | `h2 .title_h2` text | display name, may vary in case between cards |
| location line | `.detail_3` text | `"Building , District , Province"` — **parse from the right** (building names can contain commas) |
| bedrooms | `img[alt="Bedroom"]` closest `td` → `p span` | integer |
| bathrooms | `img[alt="Bathroom"]` closest `td` → `p span` | integer |
| floor | `img[alt="Floors"]` closest `td` → `p span` | integer; sometimes empty |
| size | the `td` whose `p` text ends with `sqm.` → `span` | icon `alt` is empty, so match on the text, not the icon |
| updated | `.status-update span` text | `Updated 29/10/2569` — **Buddhist calendar**, subtract 543 |
| image | `img.coverimg_2[src]` | absolute |

### Quirks that will bite a naive parser

1. `div.detail_1` contains the currency symbol `฿` and sometimes an inline `<span>Condo</span>`.
2. `div.detail_3` repeats the building name, then ` , `, then district, then ` , `, then province.
   Parse from the right: `province = parts[-1]`, `district = parts[-2]`, `building = " , ".join(parts[:-2])`.
   A 2-part line exists in the wild → `district = None` and log a warning.
3. Dates are Buddhist-era (`2569` → `2026`). Store ISO `YYYY-MM-DD`.
4. The size `<img>` has an **empty** `alt`, unlike the other three icons.
5. Card markup is not guaranteed to be well-nested enough for `BeautifulSoup`'s `td.parent` chains;
   use `select_one` scoped to the card and fall back to text matching.
6. `h2` also contains `<span>Rent</span>` — take only `.title_h2`.

---

## 3. Pagination contract

```html
<div class="col-12 text-center mt-5 numx">
  <ul class="number-page">
    <li class='active'><a>1</a></li>
    <li><a href='…&page_no=2'>2</a></li>
    <li><a href='…&page_no=2'><i class="far fa-angle-right"></i></a></li>
  </ul>
  <p>1 Of 546  </p>
</div>
```

| Field | Selector | Notes |
|---|---|---|
| current / total pages | `.number-page p` → `/^\s*(\d+)\s+Of\s+(\d+)/` | two trailing spaces; both numbers are 1-based |
| next link | `.number-page a[href*="page_no="]` | exists on every page except the last |

**Do not depend on the pager to iterate.** Build URLs yourself from the `SearchSpec` + page number
(§4). The pager is only used to know `total_pages` and to detect the last page.

### End-of-data signals (all observed)

| Situation | Response | Detection |
|---|---|---|
| `page_no` > last page | `HTTP 200`, **105 bytes**, body is a JS redirect to `/error/th` | body length < 2 KB **and** no `.condo-card` → stop, not an error |
| filter matches nothing | normal page, `1 Of 0`, zero cards | `total_units == 0` |
| page 1 of 2 with 32 results | 16 cards | — |
| last page partial | fewer than 16 cards (14 on page 546) | — |

---

## 4. Query parameters (semantics verified by counting)

| Param | Values | Meaning | Evidence |
|---|---|---|---|
| `listingtype` | `1` = sale, `2` = rent, empty = both | deal type | nav links |
| `propertytype` | `1` condo, `2` house, `4` townhouse, `5` home office, `6` commercial, `7` apartment, `8` hotel, `9` warehouse, `10` office, `11`?, `12` land, `13`? | property type | footer links |
| `bedroom` | `0` studio, `1..10`, empty = any | **exact** match, not "≥" | `bedroom=2` on the dev project → `1 Of 0` |
| `minsize` / `maxsize` | number, unit depends on `propertytype` (sqm for condo) | inclusive range | card sizes within range in samples |
| `minprice` / `maxprice` | THB per month for rent | inclusive range | card prices within range |
| `special` | `1` = "special price" nav tab | promo subset | nav |
| `sort` | `1` recommended, `2` last updated, `3` price low→high, `4` price high→low, **empty = site default** | ordering only | changing `sort` visibly reordered prices |
| `status` | `1` (hidden field posted by the site's filter forms) | **decorative — ignored on GET and POST** | `status=1` and `status=2` returned identical pages |
| `project_code` | `p00348`, `s00053`, `z00015`, `g00001`, `v1`… | project/building filter | breadcrumb links |
| `page_no` | `1..total_pages` | pagination | — |

Canonical parameter order used by the site (keep it, so URLs are diff-friendly):

```text
project_code, listingtype, propertytype, bedroom, minsize, maxsize,
minprice, maxprice, special, sort, page_no
```

Keep empty params **present but empty** (`&bedroom=&minsize=`). The site tolerates both forms, but
present-but-empty keeps our generated URLs byte-identical to the ones a human copies from the
browser, which makes bug reports trivial.

---

## 5. Can the server filter by availability? No.

The site's own filter dropdowns contain `<input type="hidden" name="status" value="1">` and POST to
`""`. Posting that form does not filter anything (see §1). **Availability is only a CSS class on
the card**, so the collector must fetch pages and filter client-side. This is the single most
important architectural fact in the project.

Consequences:

- The crawl cost is proportional to how far into the list available units extend.
- The "available-first" ordering (§6) is the only lever that keeps the crawl cheap.
- The dataset must expose the tri-state, not a boolean, so that `check-availability` is never
  silently reported as available.

---

## 6. Ordering: "available first" is a correlation, not a guarantee

`sort=` empty and `sort=1` are the same default ordering. Measured distribution:

| Query | Page | States |
|---|---|---|
| FULL, `sort=` | 1, 20, 40, 50, 60, 70 | 16 / 16 available |
| FULL, `sort=` | **71** | 12 available + 4 check-availability |
| FULL, `sort=` | 72, 74, 80, 100, 120 | 0 available (all check-availability) |
| FULL, `sort=` | 300 | 0 available (all not-available) |
| FULL, `sort=` | 546 (last) | 0 available (14 check-availability) |
| DEV, `sort=1` | 1 | 4 available, 5 check, 7 not available |
| DEV, `sort=` | 1 | identical to `sort=1` |
| DEV, `sort=3` (price) | 1 | mixed states, price ascending |
| FULL, `bedroom=1` | 1 | 16 available, total 223 pages |

**Totals (measured):** FULL = `8734` units, `546` pages; available cluster ≈ **71 pages ≈ 1132
units**; DEV = `32` units, 2 pages, 4 available total.

Ordering is stable for repeated fetches of the same URL seconds apart (16/16 identical ids and
order), but it is a ranking and **will** change between runs. Therefore:

- Collect **fresh every run**; never assume yesterday's page numbers.
- Stop rule: after the first page with **0 available** cards, scan 2 more pages; if they also have
  0, stop (default). A `--full` mode scans all `total_pages` for the paranoid weekly run.
- Deduplicate by unit id anyway (ranking changes can make a unit appear twice).
- Log the per-page state histogram; a sudden shape change (e.g. page 1 no longer all-available) is
  the earliest signal that the site changed.

---

## 7. Detail pages: coordinates and canonical building identity

The detail page embeds a Google Maps literal (their key is public in the HTML; **we never use
their key — we only read numbers out of the text**):

```js
var mapOptions = { center: {lat: 13.7187616, lng: 100.5202356}, zoom: 18 }
var marker = new google.maps.Marker({ position: new google.maps.LatLng(13.7187616, 100.5202356), … })
```

Breadcrumb (`.map-nav`), which carries the canonical building name **and** its project code:

```html
<div class="map-nav">
  <a href="https://connex.in.th/" class="link_nav">Home</a> /
  <a href="https://connex.in.th/search_en?project_code=p00348&…" class="link_nav">The Bangkok Sathorn</a> /
  For Rent Condo , The Bangkok Sathorn , … , CX-84121
</div>
```

| Field | How to extract |
|---|---|
| latitude / longitude | first match of `center:\s*\{\s*lat:\s*(-?\d+(?:\.\d+)?)\s*,\s*lng:\s*(-?\d+(?:\.\d+)?)\s*\}`; fallback `LatLng\(\s*(-?[\d.]+)\s*,\s*(-?[\d.]+)\s*\)` |
| project code | first `project_code=([a-z]\d+)` inside `.map-nav` |
| canonical building name | the `.map-nav a[href*="project_code="]` link text |
| unit code | `CX-\d+` in the last breadcrumb segment |

Verified: **3 units of "The Bangkok Sathorn" returned identical coordinates** → one fetch per
building is enough (`plan/02-architecture.md` §6 explains the merge-by-project-code step).

**Do not infer state from the detail page.** A "not available" unit's detail page still contains
the word "Available" 12 times (buttons/nav). State comes only from the search card.

### Coordinate failures

Some pages will have no `center:` literal (land/odd listings, layout changes). Treat coordinates
as optional: `lat`/`lng` may be `null`; the building stays in the dataset and is shown in the UI as
"no location" instead of being dropped.

---

## 8. Cost, latency, politeness

| Measurement | Value |
|---|---|
| search page | ~0.95 s TTFB, 17 KB gzip, 167–175 KB raw |
| detail page | ~0.8–1.2 s TTFB, 24 KB gzip, ~220 KB raw |
| 6 detail requests at 0.4 s gaps | 6 × `200`, no throttling seen |
| `Cache-Control` | `no-store, no-cache` — no conditional-request savings available |
| front door | Cloudflare; `PHPSESSID` cookie set on every response but not required |
| `robots.txt` | `User-agent: * / Allow: / / Disallow: /backend` + sitemap |

**Budget:** default `1.2 s ± 0.3 s` between requests, sequential (never parallel), identifying UA
with a contact string, honours `429`/`503` with exponential backoff and `Retry-After`, aborts after
3 consecutive failures. A full first run ≈ 1100 requests ≈ 22 min; a cached nightly run ≈ 75
requests ≈ 2 min.

---

## 9. Things that look like data but are not

| Field | Why not to use it |
|---|---|
| `<meta name="description">` "Fine 8,734 Condo For Rent" | SEO copy; comma-formatted and only sometimes in sync with the pager |
| `alt` text of the card image | free-text, duplicates the breadcrumb, contains unit code and SEO words |
| `sort` param | ordering only; changing it does not change the result set |
| `status` param | ignored (see §5) |
| "Updated" date | Buddhist calendar; also means "listing edited", not "available since" |
| unit URL slug | contains SEO keywords and the unit code; **do not** parse the building from it |

---

## 10. Risk register (each risk has a test or a mitigation in a stage)

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| R1 | Card markup changes (class renames) | parser returns 0 cards | fixture tests + "0 cards on a non-last page ⇒ hard error" rule (Stage 02) |
| R2 | Availability ordering changes | crawl stops too early, misses units | stop rule = 2 extra empty pages; `--full` weekly; per-page state histogram in the log |
| R3 | Site slows down / blocks | run fails or takes hours | throttle, backoff, resume from checkpoint, hard abort + exit code 3 |
| R4 | Coordinates missing for many buildings | empty map | optional coords, cache, "no location" bucket, stats in `meta.json` |
| R5 | Building names inconsistent (`The Bangkok Sathorn` vs `THE BANGKOK SATHORN`) | duplicate bubbles | normalise for the key; merge groups that share a `project_code` (Stage 04) |
| R6 | Pagination unstable between pages | duplicate/missed units | dedupe by `CX-` id, record duplicate count in stats |
| R7 | `check-availability` units flooded in | wrong "available" claims | tri-state model; only `available` is published; `check` counted separately |
| R8 | Site changes URL scheme | links break | store the absolute URL captured at crawl time; never rebuild it |

---

## 11. Reproduction commands (for the next investigator)

```bash
cd /home/jeka/DeepseekBench/.probe       # scratch dir used during discovery

UA='Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0 Safari/537.36'

# one page, check the pager text and the state classes
curl -sS --compressed "$URL" -H "User-Agent: $UA" -o page.html
grep -o '<p>[0-9]* Of [0-9]*' page.html
grep -o 'class="\(available\|not-available\|check-availability\)"' page.html | sort | uniq -c

# coordinates of a detail page
curl -sS --compressed "$DETAIL" -H "User-Agent: $UA" | grep -o 'center: {lat: [-0-9.]*, lng: [-0-9.]*}'
```

The scratch directory is throwaway; everything needed is in `assets/fixtures/` and in this
document. **A captured page must be redacted before it is committed** (Maps key, `csrf_token`,
`tel:` links) — the procedure is in `assets/fixtures/README.md`.
