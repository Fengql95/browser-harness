# Yelp — Search Results and Business Details

Field-tested against www.yelp.com on 2026-10-23 using a Browser Use cloud browser (and `http_get`).

## Quick summary

- Plain HTTP is blocked: `http_get` on `/search` and `/biz/` URLs returned **HTTP 403** (the site is behind DataDome; the browser URL briefly gains a `dd_referrer=` param). Use a real browser.
- In the browser, every search page and business page server-renders its data as JSON inside two script tags. No CSS selectors are needed:
  - `script[data-hypernova-key*="GondolaSearch"]` (search) holds result order, sponsored flags and pagination (`totalResults`, `resultsPerPage`, `startResult`).
  - `script[data-apollo-state]` holds an Apollo cache keyed `Business:<bizId>`, `BusinessLocation:<bizId>`, `BusinessCategory:<id>` with name, rating, review count, price, categories, address.
- Search: 10 organic results per page, paginate with `&start=10,20,...`. Yelp caps every search at **240 results** (`start=230` is the last page; `start=240` returns "the page of results you requested is unavailable").
- Phone numbers for organic results are **not** on the search page. Open the business page (`/biz/<alias>`), whose Apollo `Business` entry has `phoneNumber.formatted`, full address, hours and website.

## What works

### 1. Search businesses across pages (term + location)

```python
import json, html, time
from urllib.parse import urlencode, urlparse

def yelp_search_url(term, location, start=0):
    q = {"find_desc": term, "find_loc": location}
    if start:
        q["start"] = start
    return "https://www.yelp.com/search?" + urlencode(q)

READY_JS = """(() => !window.__yelpStale
  && !!document.querySelector('script[data-apollo-state]')
  && (!%s || !!document.querySelector('script[data-hypernova-key*="GondolaSearch"]')))()"""

def open_yelp(url, first=False, tries=3):
    """Navigate and wait until this navigation's server-rendered JSON is present (Apollo,
    plus hypernova for /search). The old document is marked first, so it can never pass
    as the new page. Retries chrome-error pages, pages without the JSON, and CDP timeouts
    (all seen in testing); raises if the page never loads."""
    is_search = urlparse(url).path.startswith("/search")
    for i in range(tries):
        try:
            if first and i == 0:
                new_tab(url)
            else:
                try:
                    js("window.__yelpStale = true")
                except Exception:
                    pass
                goto_url(url)
            wait_for_load()
            if js(READY_JS % ("true" if is_search else "false")):
                return
        except Exception as e:  # e.g. "Page.navigate timed out", "Runtime.evaluate timed out"
            print(f"load attempt {i + 1} failed: {type(e).__name__}: {e}")
        time.sleep(5 * (i + 1))
    raise RuntimeError(f"Yelp page did not load after {tries} tries: {url}")

def _strip(s):
    return s.strip().removeprefix("<!--").removesuffix("-->")

def read_search_page():
    """Parse the open search page -> (organic_rows, total, per_page, start)."""
    raw = js("""(() => {
      const h = document.querySelector('script[data-hypernova-key*="GondolaSearch"]');
      const a = document.querySelector('script[data-apollo-state]');
      return {hyper: h ? h.textContent : null, apollo: a ? a.textContent : null};
    })()""")
    if not raw or not raw.get("hyper") or not raw.get("apollo"):
        return [], 0, 0, 0
    hyper = json.loads(_strip(raw["hyper"]))                   # plain JSON
    apollo = json.loads(html.unescape(_strip(raw["apollo"])))  # whole blob is HTML-entity encoded
    sp = hyper["legacyProps"]["searchAppProps"]["searchPageProps"]
    if "mainContentComponentsListProps" not in sp:             # e.g. start >= 240
        return [], 0, 0, 0
    ctx = sp["searchContext"]
    rows = []
    for comp in sp["mainContentComponentsListProps"]:
        if comp.get("type") != "searchResultSection":
            continue
        for r in comp["props"].get("searchResults") or []:
            if r.get("isAd") or not r.get("bizId"):
                continue  # sponsored results; same biz can also appear organically
            b = apollo.get("Business:" + r["bizId"]) or {}
            loc = apollo.get((b.get("location") or {}).get("__ref", ""), {}) or {}
            addr = loc.get("address") or {}
            cats = [apollo.get(c["__ref"], {}).get("title") for c in b.get("categories") or []]
            rows.append({
                "rank": ctx.get("startResult", 0) + len(rows) + 1,
                "biz_id": r["bizId"],
                "name": html.unescape(b.get("name") or ""),
                "rating": b.get("rating"),
                "review_count": b.get("reviewCount"),
                "price": (b.get("priceRange") or {}).get("display"),
                "categories": [c for c in cats if c],
                "address": addr.get("addressLine1") or None,  # None for service-area businesses
                "city": addr.get("city"),
                "neighborhoods": loc.get("neighborhoods") or [],
                "url": "https://www.yelp.com/biz/" + b["alias"] if b.get("alias") else None,
            })
    return rows, ctx.get("totalResults", 0), ctx.get("resultsPerPage", 10), ctx.get("startResult", 0)

term, location, max_pages = "pizza", "San Francisco, CA", 3
results = []
for page in range(max_pages):  # max_pages is the explicit cap; Yelp's own cap is 240 results
    open_yelp(yelp_search_url(term, location, start=page * 10), first=(page == 0))  # raises on failure
    rows, total, per_page, start = read_search_page()
    print(f"page {page + 1}: start={start} organic={len(rows)} total={total}")
    if not rows:
        break
    results.extend(rows)
    if start + per_page >= min(total, 240):
        break
    time.sleep(3)  # polite pacing between pages

print(len(results), "organic results")
for r in results[:2]:
    print(json.dumps(r, ensure_ascii=False))
```

Output (trimmed):

```
page 1: start=0 organic=10 total=240
page 2: start=10 organic=10 total=240
page 3: start=20 organic=10 total=240
30 organic results
{"rank": 1, "biz_id": "mSMZJj2pFvttWLpcDmgrEA", "name": "Tony's Pizza Napoletana", "rating": 4.2, "review_count": 9049, "price": "$$", "categories": ["Pizza", "Italian", "Cocktail Bars"], "address": "1570 Stockton St", "city": "San Francisco", "neighborhoods": ["North Beach/Telegraph Hill"], "url": "https://www.yelp.com/biz/tonys-pizza-napoletana-san-francisco"}
{"rank": 2, "biz_id": "074WalqFJbISNhaOsuuhxg", "name": "Golden Boy Pizza", "rating": 4.3, "review_count": 305, "price": "$", "categories": ["Pizza"], "address": "1447 Taraval St", "city": "San Francisco", "neighborhoods": ["Parkside"], "url": "https://www.yelp.com/biz/golden-boy-pizza-san-francisco-5"}
```

Result order below the top spot changed between runs on the same day (rank 2 was Outta Sight Pizza in an earlier run), so do not treat a single run's ranking as fixed.

The same code worked with only `term, location` changed to `plumbers`, `Austin, TX` (10 organic per page, total 240, `price: None` for every plumber). Home-service businesses often have `address: None` and only a city, because they list a service area instead of a storefront.

### 2. Business detail page (phone, full address, hours, website)

```python
import json, html, time
from urllib.parse import urlparse, unquote

def open_yelp(url, first=False, tries=3):
    """Same as block 1, for /biz/ pages: mark the old document, navigate, wait for Apollo."""
    for i in range(tries):
        try:
            if first and i == 0:
                new_tab(url)
            else:
                try:
                    js("window.__yelpStale = true")
                except Exception:
                    pass
                goto_url(url)
            wait_for_load()
            if js("!window.__yelpStale && !!document.querySelector('script[data-apollo-state]')"):
                return
        except Exception as e:
            print(f"load attempt {i + 1} failed: {type(e).__name__}: {e}")
        time.sleep(5 * (i + 1))
    raise RuntimeError(f"Yelp page did not load after {tries} tries: {url}")

def read_biz_page():
    """Parse the open /biz/<alias> page from its Apollo cache. Returns None when no
    Business entry matches the page's alias (the cache also holds other businesses)."""
    raw = js("(() => { const a = document.querySelector('script[data-apollo-state]'); return a ? a.textContent : null })()")
    if not raw:
        return None
    apollo = json.loads(html.unescape(raw.strip().removeprefix("<!--").removesuffix("-->")))
    alias = unquote(urlparse(page_info()["url"]).path.split("/biz/")[-1].strip("/"))
    bizs = [v for k, v in apollo.items() if k.startswith("Business:")]
    b = next((v for v in bizs if v.get("alias") == alias), None)
    if not b:
        return None
    loc = apollo.get((b.get("location") or {}).get("__ref", ""), {}) or {}
    addr = loc.get("address") or {}
    week = (b.get("operationHours") or {}).get("regularHoursMergedWithSpecialHoursForCurrentWeek") or []
    site = ((b.get("externalResources") or {}).get("website") or {}).get("url")
    return {
        "biz_id": b.get("encid"),
        "name": html.unescape(b.get("name") or ""),
        "rating": b.get("rating"),
        "review_count": b.get("reviewCount"),
        "price": (b.get("priceRange") or {}).get("display"),
        "phone": (b.get("phoneNumber") or {}).get("formatted"),
        "address": (addr.get("formatted") or "").replace("\n", ", ") or None,
        "postal_code": addr.get("postalCode"),
        "neighborhoods": loc.get("neighborhoods") or [],
        "categories": [apollo.get(c["__ref"], {}).get("title") for c in b.get("categories") or []],
        "website": site,
        "is_closed": b.get("isClosed"),
        "hours": {d.get("dayOfWeekShort"): d.get("hours") for d in week},
        "url": "https://www.yelp.com/biz/" + alias,
    }

urls = [
    "https://www.yelp.com/biz/tonys-pizza-napoletana-san-francisco",
    "https://www.yelp.com/biz/outta-sight-pizza-san-francisco",
]
for i, u in enumerate(urls):
    open_yelp(u, first=(i == 0))  # raises on failure
    row = read_biz_page()
    if row is None:
        raise RuntimeError(f"no Business entry for the alias of {page_info()['url']}")
    print(json.dumps(row, ensure_ascii=False))
    time.sleep(3)
```

Feed it the `url` values from block 1 to enrich search rows with phone and full address. Output (trimmed):

```
{"biz_id": "75UY4SkdvuhJ-nZTKdVaMw", "name": "Outta Sight Pizza", "rating": 4.6, "review_count": 508, "price": "$", "phone": "(415) 829-3108", "address": "422 Larkin St, San Francisco, CA 94102", "postal_code": "94102", "neighborhoods": ["Civic Center"], "categories": ["Pizza"], "website": "https://www.thatsouttasight.com", "is_closed": false, "hours": {"Mon": ["11:00 AM - 9:00 PM"], "Tue": ["11:00 AM - 9:00 PM"], ...}, "url": "https://www.yelp.com/biz/outta-sight-pizza-san-francisco"}
```

## Fields

| Field | Search page source | Detail page source | Notes |
|---|---|---|---|
| biz id | hypernova `searchResults[].bizId` | Apollo `Business.encid` | stable 22-char id |
| name | Apollo `Business.name` | same | may contain HTML entities, unescape |
| rating | Apollo `Business.rating` | same | `None` when the business has 0 reviews |
| review count | Apollo `Business.reviewCount` | same | int |
| price | Apollo `Business.priceRange.display` | same | `"$"`..`"$$$$"` or `None` |
| categories | Apollo `Business.categories[].__ref` -> `BusinessCategory.title` | same | |
| address | Apollo `BusinessLocation.address.addressLine1`, `.city` | `.address.formatted`, `.postalCode`, `.regionCode` | search page has no ZIP |
| neighborhoods | Apollo `BusinessLocation.neighborhoods` | same | list |
| phone | only for sponsored rows (`searchResultBusiness.phone`) | Apollo `Business.phoneNumber.formatted` | organic search rows have no phone |
| website | - | Apollo `Business.externalResources.website.url` | |
| hours | - | Apollo `Business.operationHours.regularHoursMergedWithSpecialHoursForCurrentWeek[]` | `dayOfWeekShort`, `hours` |
| Yelp URL | `https://www.yelp.com/biz/<Business.alias>` | page URL | |
| pagination | hypernova `searchContext.totalResults/resultsPerPage/startResult` | - | 10 per page, capped at 240 |

## Gotchas

- **Sponsored results are mixed in.** Ad sections (`componentType: "unthemed_ads"`) appear above and below the organic list; filter `isAd`. The same business can appear as an ad and organically on the same page (observed for a plumber in Austin).
- **Organic search rows have `searchResultBusiness: null`.** The business data lives only in the Apollo cache, joined by `bizId`. Ads carry their data inline instead.
- **The Apollo blob is HTML-entity encoded as a whole** (`{&quot;ROOT_QUERY&quot;...`), wrapped in `<!-- ... -->`. Unescape it before `json.loads`. The hypernova blob is plain JSON (also inside `<!-- -->`) whose string values still contain `&amp;`; do not unescape the whole hypernova blob.
- **`ranking` is not reliable.** It was set (1..10) for the pizza search but `None` for every organic row in the plumbers search, so the code above computes rank from `startResult`.
- **240-result cap.** `totalResults` reported 240 for both test searches. `start=240` returns a "results unavailable" page whose hypernova props only contain `searchExceptionProps`. Narrow the query (neighborhood, category, filters) to get more distinct businesses.
- **JSON-LD varies between browser sessions; do not depend on it.** In one cloud session, 4 of 4 search loads had a `SearchResultsPage` JSON-LD (an `ItemList` of the first 10 businesses only) plus `Product`, `BreadcrumbList` and sometimes `WebSite`/`FAQPage`, and 4 of 4 business loads had a `Restaurant` JSON-LD with `telephone`. In a second session the same day, 4 of 4 search loads had only `BreadcrumbList`/`FAQPage` and 2 of 2 business loads had only `ItemList` x2 + `BreadcrumbList`, no `Restaurant`. The Apollo cache was present on every successful load in both sessions.
- **Transient load failures.** Seen in testing: a business page landing on `chrome-error://chromewebdata/` ("This site can't be reached"), a search load whose page had no Apollo/hypernova JSON yet, and CDP timeouts (`Page.navigate timed out`, `Runtime.evaluate timed out`). Each loaded fine on retry. `open_yelp` catches these, retries with backoff, and raises after 3 failed tries instead of returning partial results. Because a timed-out navigation can leave the previous Yelp page in place, `open_yelp` sets a `window.__yelpStale` flag on the old document before navigating and only accepts a document without it.
- **A business page's Apollo cache holds other businesses too** (e.g. 3 `Business:` entries on one page, only 1 with a phone). `read_biz_page` matches on the URL alias and returns `None` on a miss rather than guessing.
- **No consent banner** appeared for US searches from the cloud browser. No login wall on search or business pages.
- **Pacing.** About 20 browser page loads in one session, spaced 3+ seconds apart, produced no CAPTCHA or block. Keep a few seconds between pages and avoid parallel tabs on Yelp.

## What does not work

- Plain HTTP returned **HTTP 403** for both `/search?...` and `/biz/<alias>` URLs with the default Python User-Agent, a truthful `browser-harness-skill-example` one, and a full Chrome one (re-checked 2026-10-23).
- Paginating past `start=230` (see cap above).
- Phone numbers for organic results from the search page alone; one business page load per business is needed.
