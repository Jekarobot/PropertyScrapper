# Fixtures — provenance

Captured **2026-11-01** from `connex.in.th` with a desktop Chrome UA
(`Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0 Safari/537.36`),
`curl --compressed`, no cookies required.

Copy these files into `pipeline/tests/fixtures/` in Stage 00. They are **frozen**: if a test fails,
fix the parser; if the site genuinely changed, re-capture, document the diff, and update the golden
snapshot in a separate commit (see `plan/91-agent-briefs.md` §6).

| File | Source URL | HTTP | Raw size | What it pins |
|---|---|---|---|---|
| `search-dev-p1.html` | `…/search_en?project_code=p00348&listingtype=2&propertytype=1&bedroom=&minsize=55&maxsize=&minprice=25000&maxprice=50000&special=&sort=1&page_no=1` | 200 | 169 109 B | 16 cards; 4 available / 5 check / 7 not available; pager `1 Of 2`; card #1 = `CX-84121`, ฿45,000, 65 sqm, 1 bed, 1 bath, floor 27, `The Bangkok Sathorn , Sa Thon , Bangkok`, updated `29/10/2569` |
| `search-dev-p2.html` | same URL with `page_no=2` | 200 | 169 349 B | 16 cards; 0 available / 1 check / 15 not available; pager `2 Of 2` |
| `search-boundary-p71.html` | `…/search_en?project_code=&listingtype=2&propertytype=1&bedroom=&minsize=45&maxsize=150&minprice=25000&maxprice=50000&special=&sort=&page_no=71` | 200 | 174 631 B | 12 available + 4 check on one page; pager `71 Of 546` — the mixed page that makes `empty_streak` necessary |
| `search-error-out-of-range.html` | `…&page_no=3` on the 2-page DEV query | 200 | 105 B | the JS-redirect stub to `/error/th` — end-of-data signal |
| `detail-cx84121.html` | `https://connex.in.th/en/detail/for-rent-condo-the-bangkok-sathorn-high-floor-river-view-bts-surasak-yannawa-sa-thon-bangkok-cx-84121` | 200 | 222 243 B | `center: {lat: 13.7187616, lng: 100.5202356}`; breadcrumb project `p00348` and name `The Bangkok Sathorn`; contains the word "Available" even though state must come from the search page |

Notes:

- Sizes drift by a few hundred bytes between fetches (ad/analytics blocks); tests must never assert
  byte sizes. The table is for the reviewer's sanity check only.
- `detail-cx84121.html` was fetched without `--compressed`; the file is UTF-8.
- No personal data is present in these files (listing data only, no contact forms submitted).
- Synthetic fixtures are produced inside tests (string edits), not stored here; each stage doc says
  exactly which edit creates which case.
