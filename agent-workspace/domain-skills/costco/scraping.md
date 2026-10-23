# Costco — Search Results & Product Details

Field-tested against www.costco.com on 2026-10-23 using a Browser Use cloud browser (search) and plain Python `urllib` (product pages).

## Quick summary

- **Search results are not in the HTML.** `GET https://www.costco.com/s?keyword=...` from plain Python returned 200 (2.4 MB) with zero product ids and no JSON-LD. The grid is filled client-side by `POST https://gdx-api.costco.com/catalog/search/api/v1/search`, which is protected by an anti-bot SDK (Kasada, `x-kpsdk-*` headers). From plain Python that POST returned **429** with an empty body.
- **Inside a loaded search page it works.** Record the page's own search request (body carries its visitor id and the location it picked for the browser), then re-send it with `fetch()` from the page with a different `offset`. The page's fetch wrapper adds fresh anti-bot tokens. Each response has 24 results (title, brands, rating, rating count, URL, variant ids) plus an `inventoryResponse` with prices and delivery status.
- **Product pages work with plain HTTP.** `GET https://www.costco.com/p/-/<slug>/<product_id>` returned 200 (about 2.7 MB) with a schema.org `Product` JSON-LD block (name, item number as `sku`, brand, rating, review count, price) and a `BreadcrumbList`, plus server-rendered features and a specs table. 24 of 24 product pages fetched at 2 s spacing returned 200 with JSON-LD.
- **Some prices are member-only.** Items flagged `disp_price_in_cart_only` show "Sign In for Details" on search tiles and "Sign In for Price" on the product page when logged out. Both blocks return `price: null` for those, matching what a logged-out visitor sees.

## What works

### 1. Search, 2+ pages (browser, the page's own search API)

```python
import email.utils, json, time, urllib.parse
from datetime import datetime, timezone

QUERY = "air fryer"   # any search term
MAX_PAGES = 3         # safety cap; the site shows 24 results per page. Reported when hit.
SEARCH_API = "https://gdx-api.costco.com/catalog/search/api/v1/search"
SEARCH_URL = "https://www.costco.com/s?" + urllib.parse.urlencode({"keyword": QUERY})

# Search results are not in the HTML. The page POSTs to the search API with a body that
# carries its visitor id and the location it picked for this browser.
# Record that request (before the page loads), then re-send it from the page with other offsets.
RECORDER = """(() => {
  window.__costcoSearch = [];
  const orig = window.fetch;
  window.fetch = async function(input, init) {
    const url = typeof input === 'string' ? input : (input && input.url) || String(input);
    const isSearch = url.includes('/catalog/search/api/v1/search');
    let body = init && init.body, headers = (init && init.headers) || {};
    if (isSearch && input instanceof Request) {   // the anti-bot wrapper passes a Request object
      body = await input.clone().text();
      headers = input.headers;
    }
    const res = await orig.apply(this, arguments);
    if (isSearch && typeof body === 'string') {
      if (headers instanceof Headers) headers = Object.fromEntries(headers.entries());
      window.__costcoSearch.push({status: res.status, headers, body});
    }
    return res;
  };
})()"""

def js_retry(expr, tries=3):
    # js() reports a CDP timeout as RuntimeError("Runtime.evaluate timed out ..."); raw cdp raises TimeoutError.
    for i in range(tries):
        try:
            return js(expr)
        except TimeoutError:
            pass
        except RuntimeError as e:
            if "timed out" not in str(e):
                raise
        time.sleep(2)
    raise TimeoutError(f"js() timed out {tries} times: {expr[:80]}")

def retry_after_seconds(value):
    """Retry-After is either seconds or an HTTP date."""
    if not value:
        return None
    try:
        return max(0.0, float(value))
    except ValueError:
        pass
    try:
        when = email.utils.parsedate_to_datetime(value)
    except (TypeError, ValueError):
        return None
    if when.tzinfo is None:
        when = when.replace(tzinfo=timezone.utc)
    return max(0.0, (when - datetime.now(timezone.utc)).total_seconds())

def query_of(call):
    try:
        return str(json.loads(call.get("body") or "{}").get("query", "")).strip().lower()
    except (ValueError, AttributeError):
        return None

tab = new_tab()   # blank background tab; never brought to the front
try:
    cdp("Page.addScriptToEvaluateOnNewDocument", source=RECORDER)
    goto_url(SEARCH_URL)
    wait_for_load()

    # Confirm this document is the requested search page, then take its first successful search call.
    first, url = None, None
    for _ in range(30):
        url = js_retry("location.href") or ""
        q = urllib.parse.parse_qs(urllib.parse.urlsplit(url).query)
        if url.startswith("https://www.costco.com/s?") and q.get("keyword") == [QUERY]:
            calls = js_retry("window.__costcoSearch || []") or []
            first = next((c for c in calls if c.get("status") == 200 and query_of(c) == QUERY.lower()), None)
            if first:
                break
        time.sleep(1)
    if not first:
        raise SystemExit(f"no search API call for {QUERY!r} recorded; page is {url}")

    # x-kpsdk-* are one-time anti-bot tokens; the page's own fetch wrapper adds fresh ones to every call.
    headers = {k: v for k, v in first["headers"].items() if not k.lower().startswith("x-kpsdk")}
    base_body = json.loads(first["body"])
    page_size = base_body["pageSize"] if isinstance(base_body.get("pageSize"), int) and base_body["pageSize"] > 0 else 24

    def fetch_search(offset, tries=4):
        """POST from the page without blocking js(), poll for the result, retry 429/5xx/network errors."""
        body = json.dumps(dict(base_body, offset=offset))
        status = None
        for attempt in range(tries):
            key = json.dumps(f"o{offset}a{attempt}")
            js_retry("(() => { const s = window.__costcoReplay = window.__costcoReplay || {};"
                     " const k = " + key + "; if (k in s) return true; s[k] = null;"
                     " fetch(" + json.dumps(SEARCH_API) + ", {method: 'POST', headers: " + json.dumps(headers)
                     + ", body: " + json.dumps(body) + "})"
                     ".then(async r => { s[k] = {status: r.status, retryAfter: r.headers.get('retry-after'), body: await r.text()}; })"
                     ".catch(e => { s[k] = {status: 0, retryAfter: null, body: String(e)}; }); return true; })()")
            r, deadline = None, time.time() + 45
            while not r and time.time() < deadline:
                time.sleep(0.5)
                r = js_retry("(window.__costcoReplay || {})[" + key + "]")
            js_retry("delete (window.__costcoReplay || {})[" + key + "]")
            status = r["status"] if r else None   # None = no answer within 45 s
            if status == 200:
                return json.loads(r["body"])
            if status not in (None, 0, 429) and status < 500:
                raise RuntimeError(f"search HTTP {status} at offset {offset}: {r['body'][:200]}")
            wait = retry_after_seconds(r.get("retryAfter")) if r else None
            if attempt < tries - 1:
                time.sleep(wait if wait is not None else 2 ** (attempt + 1))
        raise RuntimeError(f"search failed at offset {offset} after {tries} tries (last status {status})")

    def attr(attrs, key):
        a = attrs.get(key) if isinstance(attrs.get(key), dict) else {}
        return (a.get("text") or []) + (a.get("numbers") or [])

    def price_text(p):
        if not isinstance(p, dict):
            return None
        lo, hi = p.get("minPrice"), p.get("maxPrice")
        if not isinstance(lo, (str, int, float)) or lo in ("", 0):
            return None
        if not isinstance(hi, (str, int, float)) or hi in ("", 0):
            hi = lo
        return f"${lo}" if lo == hi else f"${lo} - ${hi}"

    def parse(result, inventory):
        p = result.get("product") if isinstance(result.get("product"), dict) else {}
        attrs = p.get("attributes") if isinstance(p.get("attributes"), dict) else {}
        inv = inventory.get(result.get("id")) or {}
        rating = p.get("rating") if isinstance(p.get("rating"), dict) else {}
        avg, count = rating.get("averageRating"), rating.get("ratingCount")
        # Tiles with disp_price_in_cart_only == 1 show "Sign In for Details" instead of a price.
        hidden = 1.0 in attr(attrs, "disp_price_in_cart_only")
        variants = [v for v in p.get("variants") or [] if isinstance(v, dict)]
        return {
            "product_id": result.get("id"),
            "item_numbers": [v["id"] for v in variants if v.get("id")],   # "Item 1234567" on the product page
            "title": p.get("title"),
            "brand": (p.get("brands") or [None])[0],
            "price": None if hidden else price_text(inv.get("deliveryPrice")),
            "price_note": "Sign In for Details" if hidden else None,
            "rating": round(avg, 2) if isinstance(avg, (int, float)) and avg else None,
            "review_count": count if isinstance(count, int) else None,
            "url": p.get("uri"),
            "category": (p.get("categories") or [None])[-1],
            "pills": attr(attrs, "pills"),
            "delivery": inv.get("deliveryAvailability"),
        }

    rows, seen, total, complete = [], set(), None, False
    for page in range(MAX_PAGES):
        data = fetch_search(page * page_size)
        sr = data.get("searchResult") or {}
        total = sr.get("totalSize")
        results = [r for r in sr.get("results") or [] if isinstance(r, dict)]
        inventory = {i.get("productId"): i for i in data.get("inventoryResponse") or [] if isinstance(i, dict)}
        for res in results:
            row = parse(res, inventory)
            if row["product_id"] is not None:   # dedupe by id; keep rows without one
                if row["product_id"] in seen:
                    continue
                seen.add(row["product_id"])
            rows.append(row)
        print(f"page {page + 1}: {len(results)} results, {len(rows)} unique so far (totalSize {total})")
        if len(results) < page_size:   # only a short successful page ends the search
            complete = True
            break
        time.sleep(2)   # polite pacing
    if not complete:
        print(f"MAX_PAGES cap hit: stopped after {MAX_PAGES} full pages, more results may remain (totalSize {total})")

    print("price shown:", sum(1 for r in rows if r["price"]),
          "| sign in for details:", sum(1 for r in rows if r["price_note"]),
          "| no online price:", sum(1 for r in rows if not r["price"] and not r["price_note"]))
    print(json.dumps(rows[:3], indent=1))
finally:
    close_tab(tab)
```

Observed output (trimmed):

```
page 1: 24 results, 24 unique so far (totalSize 255)
page 2: 24 results, 48 unique so far (totalSize 255)
page 3: 24 results, 72 unique so far (totalSize 255)
MAX_PAGES cap hit: stopped after 3 full pages, more results may remain (totalSize 255)
price shown: 12 | sign in for details: 60 | no online price: 0
[
 {
  "product_id": "4000376434",
  "item_numbers": ["1898552"],
  "title": "Ninja Crispi 5-in-1 Portable Glass Air Fry Cooking System, 3 Containers & Cover",
  "brand": "Ninja",
  "price": null,
  "price_note": "Sign In for Details",
  "rating": 4.66,
  "review_count": 2706,
  "url": "https://www.costco.com/p/-/ninja-crispi-5-in-1-portable-glass-air-fry-cooking-system-3-containers--cover/4000376434",
  "category": "Appliances > Small Kitchen Appliances > Air Fryers",
  "pills": ["Instant Savings"],
  "delivery": "IN_STOCK"
 },
 {
  "product_id": "4201032285",
  "item_numbers": ["7232432"],
  "title": "Gourmia 9-Qt Digital Air Fryer with Ceramic Coating",
  "brand": "Gourmia",
  "price": "$59.99",
  "price_note": null,
  "rating": 4.4,
  "review_count": 5,
  "url": "https://www.costco.com/p/-/gourmia-9-qt-digital-air-fryer-with-ceramic-coating/4201032285",
  "category": "Appliances > Small Kitchen Appliances > Air Fryers",
  "pills": ["New"],
  "delivery": "IN_STOCK"
 },
 ...
]
```

The same block with `QUERY = "paper towels"` gave 24/48/72 unique rows over 3 pages (totalSize 888 to 891), with a price shown on 70 of 72. Big appliances are mostly member-only, groceries mostly are not. With `QUERY = "olive oil"` and `MAX_PAGES = 10` it ran to the end on its own: pages 1 to 6 had 24 results, page 7 had 15 (totalSize 159), 151 unique rows, no cap message.

### 2. Product details (plain HTTP, no browser)

```python
import email.utils, html, json, re, sys, time, urllib.error, urllib.request
from datetime import datetime, timezone

# Product URLs from the search rows (url field). Any https://www.costco.com/p/-/<slug>/<product_id>
# or <slug>.product.<product_id>.html URL works.
URLS = [
    "https://www.costco.com/p/-/t-fal-dual-window-infrared-air-fryer-73qt-dual-heating/4000430818",
    "https://www.costco.com/p/-/gourmia-9-qt-digital-air-fryer-with-ceramic-coating/4201032285",
    "https://www.costco.com/p/-/cuisinart-extra-large-26l-10-in-1-digital-air-fryer-oven/4000398962",
]
# A plain, truthful client UA works here; no browser UA needed.
HEADERS = {"User-Agent": f"Python-urllib/{sys.version_info[0]}.{sys.version_info[1]}", "Accept-Language": "en-US,en;q=0.9"}
RETRY_STATUS = {429, 500, 502, 503, 504}

def retry_after_seconds(value):
    """Retry-After is either seconds or an HTTP date."""
    if not value:
        return None
    try:
        return max(0.0, float(value))
    except ValueError:
        pass
    try:
        when = email.utils.parsedate_to_datetime(value)
    except (TypeError, ValueError):
        return None
    if when.tzinfo is None:
        when = when.replace(tzinfo=timezone.utc)
    return max(0.0, (when - datetime.now(timezone.utc)).total_seconds())

def get_page(url, tries=4):
    """(status, html or None). Retries 429/5xx/network errors with backoff and raises if they persist."""
    last = None
    for attempt in range(tries):
        wait = None
        try:
            with urllib.request.urlopen(urllib.request.Request(url, headers=HEADERS), timeout=30) as r:
                return r.status, r.read().decode("utf-8", "replace")
        except urllib.error.HTTPError as e:
            if e.code not in RETRY_STATUS:
                return e.code, None   # 404 = unknown or removed product
            wait, last = retry_after_seconds(e.headers.get("Retry-After")), f"HTTP {e.code}"
        except (urllib.error.URLError, TimeoutError, ConnectionError) as e:
            last = repr(e)
        if attempt < tries - 1:
            time.sleep(wait if wait is not None else 2 ** (attempt + 1))
    raise RuntimeError(f"{url}: failed after {tries} tries ({last})")

def clean(fragment):
    return " ".join(html.unescape(re.sub(r"<[^>]+>", " ", fragment)).split())

def first_dict(v):
    if isinstance(v, list):
        v = next((x for x in v if isinstance(x, dict)), None)
    return v if isinstance(v, dict) else {}

def image_url(img):
    if isinstance(img, list):
        img = img[0] if img else None
    if isinstance(img, dict):   # schema.org ImageObject
        img = img.get("url") or img.get("contentUrl")
    return img if isinstance(img, str) else None

def product_details(url):
    status, page = get_page(url)
    if page is None:
        return {"url": url, "error": f"HTTP {status}"}
    ld = {}
    for block in re.findall(r'<script[^>]*application/ld\+json[^>]*>(.*?)</script>', page, re.S):
        try:
            data = json.loads(block)
        except ValueError:
            continue
        objs = data if isinstance(data, list) else (data.get("@graph") or [data]) if isinstance(data, dict) else []
        for obj in objs:
            if not isinstance(obj, dict):
                continue
            types = obj.get("@type") if isinstance(obj.get("@type"), list) else [obj.get("@type")]
            for t in types:
                if isinstance(t, str):
                    ld.setdefault(t, obj)
    prod = ld.get("Product")
    if not prod:
        return {"url": url, "error": "no Product JSON-LD (not a product page)"}
    offer = first_dict(prod.get("offers"))
    rating = first_dict(prod.get("aggregateRating"))
    brand = prod.get("brand")
    # Same rule as the search tiles: dispPriceInCartOnly 1 -> the page shows "Sign In for Price".
    # The first match is the main item (itemSequence 0); later ones are bundled or related items.
    flag = re.search(r'dispPriceInCartOnly\\":(\d)', page)
    hidden = bool(flag and flag.group(1) == "1")
    item_line = re.search(r'data-testid="Text_item-number">(.*?)</div>', page)   # "Item 1994092 | Model EY8338U0"
    model = re.search(r"Model\s+(\S+)", clean(item_line.group(1))) if item_line else None
    specs = {}
    for k, v in re.findall(r'<th[^>]*id="ProductSpecifications-th-row\d+-0"[^>]*>(.*?)</th><td[^>]*>(.*?)</td>', page):
        specs.setdefault(clean(k), []).append(clean(v))   # some keys repeat (e.g. two Capacity rows)
    crumbs = first_dict(ld.get("BreadcrumbList")).get("itemListElement") or []
    desc = prod.get("description")
    return {
        "url": prod.get("url") if isinstance(prod.get("url"), str) else url,
        "title": prod.get("name"),
        "item_number": prod.get("sku"),
        "model": model.group(1) if model else None,
        "brand": brand.get("name") if isinstance(brand, dict) else brand if isinstance(brand, str) else None,
        "price": None if hidden else offer.get("price"),
        "price_note": "Sign In for Price" if hidden else None,
        "currency": offer.get("priceCurrency"),
        "rating": rating.get("ratingValue"),
        "review_count": rating.get("ratingCount"),
        "breadcrumbs": [e.get("name") for e in crumbs if isinstance(e, dict)],
        "features": [clean(f) for f in re.findall(r'data-testid="Text_feature-\d+">(.*?)</div>', page)],
        "specs": {k: "; ".join(v) for k, v in specs.items()},
        "description": desc if isinstance(desc, str) else None,
        "image": image_url(prod.get("image")),
    }

for i, url in enumerate(URLS):
    if i:
        time.sleep(2)   # polite pacing
    d = product_details(url)
    if d.get("error"):
        print(url, d["error"])
        continue
    print(json.dumps({**d, "specs": dict(list(d["specs"].items())[:5]), "description": (d["description"] or "")[:120]}, indent=1))
```

Observed output (trimmed, second product):

```
{
 "url": "https://www.costco.com/p/-/gourmia-9-qt-digital-air-fryer-with-ceramic-coating/4201032285",
 "title": "Gourmia 9-Qt Digital Air Fryer with Ceramic Coating",
 "item_number": "7232432",
 "model": "GAF998",
 "brand": "Gourmia",
 "price": 59.99,
 "price_note": null,
 "currency": "USD",
 "rating": 4.4,
 "review_count": 5,
 "breadcrumbs": ["Home", "Appliances", "Small Kitchen Appliances", "Air Fryers"],
 "features": ["Ceramic-Coated Basket and Crisper Tray", "9 Cooking Presets Plus 3 Customizable Favorites Presets", ...],
 "specs": {"Brand": "Gourmia", "Capacity": "9 qt.", "Color": "Black", "Cord Length": "31.5 in.", "Dimensions - Depth": "16.2 in."},
 "description": "Gourmia 9-Qt Digital Air Fryer with Ceramic Coating, Ceramic-Coated Basket and Crisper Tray, 9 Cooking Presets Plus 3 Cu",
 "image": "https://gdx-assets.costco.com/adobe/assets/urn:aaid:aem:859d2718-.../as/4201032285-847__1.avif"
}
```

The T-fal item came back with `"price": null, "price_note": "Sign In for Price"`, which is what its live page showed logged out.

## Fields

Search (block 1), one row per product. `p` = `searchResult.results[i].product`, `inv` = the `inventoryResponse` entry with the same `productId`.

| Field | Source | Example / notes |
|---|---|---|
| product_id | `results[i].id` | `"4201032285"`, the number at the end of product URLs |
| item_numbers | `p.variants[].id` | `["7232432"]`, shown as "Item 7232432" on the product page; multi-variant products list one per color/size |
| title | `p.title` | |
| brand | `p.brands[0]` | |
| price | `inv.deliveryPrice.minPrice` / `maxPrice` | `"$59.99"`; a range `"$lo - $hi"` when variants differ; `null` for member-only items; matched the tile on all 24 page-1 tiles checked |
| price_note | `p.attributes.disp_price_in_cart_only.numbers == [1.0]` | `"Sign In for Details"` (tile shows no price), else `null` |
| rating | `p.rating.averageRating`, rounded to 2 places | `4.4` |
| review_count | `p.rating.ratingCount` | `5` |
| url | `p.uri` | `https://www.costco.com/p/-/<slug>/<product_id>`; the tiles link to the equivalent `<slug>.product.<product_id>.html` |
| category | `p.categories[-1]` | `"Appliances > Small Kitchen Appliances > Air Fryers"` |
| pills | `p.attributes.pills.text` | `["Online Only"]`, `["Members Only"]`, `["New"]`, ... |
| delivery | `inv.deliveryAvailability` | `"IN_STOCK"` |

Not returned by block 1 but in the same response: `p.attributes.model.text`, `primary_image.text`, `marketing_statement.text`; `inv.warehousePrice`, `inv.originalPrice` (warehouse price is for the store picked for the browser's location); `searchResult.totalSize` (255 for "air fryer", printed per page, drifts by a few between calls).

Product page (block 2), one dict per URL. On a 404 or a page without `Product` JSON-LD it returns `{"url", "error"}` instead.

| Field | Source | Example / notes |
|---|---|---|
| url | JSON-LD `Product.url`, else the requested URL | |
| title | `Product.name` | |
| item_number | `Product.sku` | `"7232432"` |
| model | `data-testid="Text_item-number"` text after "Model" | `"GAF998"` |
| brand | `Product.brand.name` (or a plain string) | `"Gourmia"` |
| price | `Product.offers.price` | `59.99`; `null` for member-only items |
| price_note | `dispPriceInCartOnly` of the main item | `"Sign In for Price"`, else `null` |
| currency | `Product.offers.priceCurrency` | `"USD"` |
| rating | `Product.aggregateRating.ratingValue` | `4.4` |
| review_count | `Product.aggregateRating.ratingCount` | `5` |
| breadcrumbs | `BreadcrumbList.itemListElement[].name` | `["Home", "Appliances", ...]` |
| features | `data-testid="Text_feature-N"` bullets | list of strings |
| specs | `table#ProductSpecifications` rows | dict; repeated keys joined with `"; "` |
| description | `Product.description` | |
| image | `Product.image` (string, list or ImageObject) | one URL string |

Also on the page but not returned: JSON-LD `offers.availability` (unreliable, see Gotchas) and the embedded `displayPrice` (`onlinePrice`, `aggregatedDiscountAmt`, `deliveredPrice`).

## Gotchas

- **Install the recorder before the page loads.** The search call fires during page load, so `Page.addScriptToEvaluateOnNewDocument` must run on a blank tab before `goto_url`. The anti-bot SDK wraps `fetch` after the recorder and calls it with a `Request` object; a first version that read only `init.body` recorded nothing, so read the body from `input.clone()`.
- **Do not hand-build the search body.** In-page, a minimal body (`query`, `pageSize`, `offset`, `searchMode`) returned 400, and the same without the `client-identifier` header returned 401 "Missing credentials". Copy the recorded body and headers and change only `offset`.
- **Page size.** The site sends `pageSize: 24`. `pageSize` 48 and 96 also returned full pages in-page. An `offset` past the end (960 and 2400 for a query with totalSize about 890) returned 200 with 0 results. Block 1 stops only on a short or empty successful page, not on `totalSize`, because `totalSize` drifts.
- **Failures are not end of results.** Block 1 retries a 429, 5xx, network error or no answer within 45 s up to 4 times (honouring `Retry-After` in seconds or as a date, else 2/4/8 s backoff) and then raises; other 4xx raise at once. Block 2 does the same for product pages and returns an error record only for a definite 404 or a page without `Product` JSON-LD. No 429 or 5xx was seen today, so the retry paths are untested against the live site.
- **`js()` timeouts.** In this harness `js()` gives up after about 5 s and raises `RuntimeError("Runtime.evaluate timed out ...")`, not `TimeoutError`. Block 1 therefore starts each search POST without awaiting it and polls a page variable, and `js_retry` retries only timeout errors.
- **Counts drift and repeat.** `totalSize` changed between calls for the same query (255/252, 891/888) and products repeat across pages: one across pages 1 and 2 of "air fryer" in one run (71 unique of 72), and 8 on pages 5 and 6 of "olive oil" (151 unique of 159). Block 1 dedupes by `product_id`.
- **Out-of-stock items are hidden by default.** The recorded body had `"filterBy": ["HIDE_OUT_OF_STOCK"]`, matching the unchecked "Show Out of Stock Items" filter.
- **Results depend on location.** On load the site picks a delivery ZIP and a "My Warehouse" store from the browser's IP and puts them in the request body (`warehouseId`, `shipToPostal`, `deliveryLocations`). Prices, `warehousePrice` and availability are for that location.
- **Plain-HTTP availability is not real.** JSON-LD `offers.availability` was `OutOfStock` on all 5 product pages inspected over plain HTTP, including items the search API and the live page showed as available. Use `delivery` from block 1 for stock.
- **JSON-LD price is the online list price.** For a member-only item with a $20 discount, JSON-LD said 99.99 while `displayPrice.deliveredPrice` was 79.99. Block 2 nulls the price for member-only items anyway.
- **Review counts lag a little.** One live product page showed 275 reviews while JSON-LD and search said 272. Treat counts as approximate.
- **User-Agent.** Product pages returned 200 with the default `Python-urllib` UA and with a custom tool UA, so block 2 sends a truthful `Python-urllib/3.x` UA. No browser UA is needed on the plain-HTTP path.
- **Pagination URL.** `https://www.costco.com/s?keyword=air+fryer&currentPage=2` loaded page 2 (the page sent `offset: 24`), useful if you prefer navigating to replaying.
- **No consent prompt** got in the way from the cloud browser's US exit IP; search tiles and product pages rendered without any click.
- **Background tab is enough.** Block 1 runs in a background tab and never activates it; search calls fired and replays worked without bringing the tab to the front.
- **DOM fallback.** Product links on search tiles match `a[href*=".product."]`, and the pager buttons have `aria-label="Go to page 2"`, `"Go to next page"`. Prefer the JSON.
- **Pacing.** About 20 search API calls at 2 s spacing and 24 product pages over plain HTTP at 2 s spacing all returned 200, with no 429 or challenge. Product pages are about 2.7 MB each, so fetch only what you need. Heavier rates were not tested.

## What does not work

- **Plain HTTP search**: `https://www.costco.com/s?keyword=air+fryer` returned 200 but no product ids, ratings or prices in the HTML; results load client-side.
- **Plain HTTP to the APIs**: `POST gdx-api.costco.com/catalog/search/api/v1/search` returned **429** with an empty body (anti-bot challenge), and `GET gdx-api.costco.com/catalog/product/product-api/v2/products/summary` returned **401** "Missing credentials". Use the browser for search instead of trying to get around this.
- **Member-only prices**: items flagged `disp_price_in_cart_only` show "Sign In for Details" / "Sign In for Price" logged out. This skill does not sign in and does not report those prices.
- **JSON-LD on search pages**: none.
- **Unknown product ids**: `/p/-/x/<bad id>` and `<slug>.product.<bad id>.html` returned 404.
