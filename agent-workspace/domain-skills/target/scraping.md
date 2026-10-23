# Target — Search Results & Product Details

Field-tested against www.target.com on 2026-10-23 using a Browser Use cloud browser (search, live PDP price) and http_get / plain urllib (product details).

## Quick summary

- **Search results** are not in the HTML. The grid is filled client-side by `GET https://cdui-orchestrations.target.com/cdui_orchestrations/v1/pages/slp?...`. Calling that endpoint from plain Python returned **HTTP 435** with a PerimeterX block payload. From inside a loaded search page, `fetch()` of the same URL returned **200** with clean JSON (title, price, rating, rating count, TCIN, URL, shipping/pickup status). Paginate by changing `offset` (24 per page).
- **Product details** (`/p/-/A-<TCIN>`): plain HTTP GET returned **200** with `__NEXT_DATA__` holding the full product record (title, brand, DPCI, UPC, breadcrumbs, rating distribution, review/question counts, bullets, description, images). Price and fulfillment are **not** in that HTML (deferred modules).
- **Live price + fulfillment on a product page**: open it in the browser, bring the tab to the front, read two `data-test` modules.
- No JSON-LD product data on either page type (search has only a `WebPage` breadcrumb block, product pages had none).
- **A fresh browser can be blocked on its very first search.** In 1 of 3 fresh browser sessions on 2026-10-23 PerimeterX answered the page's own slp calls with 435 before any extra request, and the page read "We couldn't find a match for your search." Block 1 detects this and stops; a new browser session (new exit IP) worked.

## What works

### 1. Search, 2+ pages (browser, in-page fetch of the site's own API)

```python
import html, json, time, urllib.parse

QUERY = "air fryer"   # any search term
MAX_PAGES = 3         # the site asks for 24 organic results per page

BLOCKED = ("PerimeterX blocked this browser session (slp HTTP 435). The page then shows "
           "\"We couldn't find a match for your search.\" Do not retry in this session: "
           "start a new browser session (new exit IP) and run again.")

tab = new_tab("https://www.target.com/s?" + urllib.parse.urlencode({"searchTerm": QUERY}))
wait_for_load()

# The grid is filled client-side by GET cdui-orchestrations.target.com/.../pages/slp.
# Reuse the exact URL the page just sent: it already carries the public web key,
# visitor id and the store chosen for this browser's location.
entry = None
for _ in range(20):
    entry = js("performance.getEntriesByType('resource')"
               ".filter(e => e.name.includes('/cdui_orchestrations/v1/pages/slp'))"
               ".map(e => ({url: e.name, status: e.responseStatus})).pop() || null")
    if entry:
        break
    time.sleep(1)
if not entry:
    raise SystemExit(f"no slp call seen, check the page: {page_info()}")
if entry["status"] == 435:
    raise SystemExit(BLOCKED)
slp_url = entry["url"]

def fetch_search(offset):
    parts = urllib.parse.urlsplit(slp_url)
    q = dict(urllib.parse.parse_qsl(parts.query, keep_blank_values=True))
    q["offset"] = str(offset)
    qs = {"searchTerm": QUERY}
    if offset:
        qs["Nao"] = str(offset)          # what the page itself sends for page 2+
    q["query_string"] = urllib.parse.urlencode(qs)
    url = urllib.parse.urlunsplit(parts._replace(query=urllib.parse.urlencode(q)))
    r = js("(async u => { const r = await fetch(u, {credentials: 'include'});"
           " return {status: r.status, body: await r.text()}; })(" + json.dumps(url) + ")")
    if r["status"] == 435:
        raise SystemExit(BLOCKED)
    if r["status"] != 200:
        raise RuntimeError(f"slp HTTP {r['status']}: {r['body'][:200]}")
    for m in json.loads(r["body"]).get("data_source_modules") or []:
        sr = (m.get("module_data") or {}).get("search_response")
        if sr:
            return sr
    return {}

def parse(p):
    item = p.get("item") or {}
    parent = p.get("parent") or {}
    price = p.get("price") or {}
    stats = (p.get("ratings_and_reviews") or parent.get("ratings_and_reviews") or {}).get("statistics") or {}
    rating = stats.get("rating") or {}
    ful = p.get("fulfillment") or {}
    stores = ful.get("store_options") or [{}]
    return {
        "tcin": p.get("tcin"),
        "title": html.unescape((item.get("product_description") or {}).get("title") or ""),
        "brand": (item.get("primary_brand") or {}).get("name"),
        "price": price.get("formatted_current_price"),           # "$59.99", "$49.99 - $59.99" or "See price in cart"
        "price_value": price.get("current_retail"),              # float, use .get: not every price object has it
        "reg_price": price.get("reg_retail"),
        "rating": rating.get("average"),
        "rating_count": rating.get("count"),
        "url": (item.get("enrichment") or {}).get("buy_url"),
        "ship_status": (ful.get("shipping_options") or {}).get("availability_status"),
        "pickup_status": (stores[0].get("order_pickup") or {}).get("availability_status"),
        "sold_out": ful.get("sold_out"),
        "sponsored": bool(p.get("is_sponsored_sku")),
    }

rows, seen, meta = [], set(), {}
for page in range(MAX_PAGES):
    sr = fetch_search(page * 24)
    meta = (sr.get("search_response") or {}).get("metadata") or {}
    products = sr.get("products") or []
    if not products:
        break
    for p in products:
        row = parse(p)
        if row["sponsored"] or row["tcin"] in seen:   # sponsored slots and cross-page repeats
            continue
        seen.add(row["tcin"])
        rows.append(row)
    print(f"page {meta.get('current_page')}/{meta.get('total_pages')}: {len(products)} products, {len(rows)} unique so far")
    if meta.get("current_page") and meta.get("current_page") >= (meta.get("total_pages") or 0):
        break
    time.sleep(2)   # polite pacing

print("total_results:", meta.get("total_results"))
print(json.dumps(rows[:2], indent=1))
close_tab(tab)
```

Observed output (trimmed):

```
page 1/12: 30 products, 24 unique so far
page 2/12: 30 products, 48 unique so far
page 3/12: 30 products, 72 unique so far
total_results: 276
[
 {
  "tcin": "89110863",
  "title": "Gourmia 6qt Digital Window Air Fryer with 12 Presets & Guided Cooking Black: Stainless Steel, 1-Year Warranty",
  "brand": "Gourmia",
  "price": "$59.99",
  "price_value": 59.99,
  "reg_price": 69.99,
  "rating": 4.07,
  "rating_count": 2296,
  "url": "https://www.target.com/p/gourmia-6-qt-digital-window-air-fryer-with-12-presets-38-guided-cooking-black/-/A-89110863",
  "ship_status": "IN_STOCK",
  "pickup_status": "IN_STOCK",
  "sold_out": false,
  "sponsored": false
 },
 ...
]
```

For "air fryer" each page returned 30 products: 24 organic plus 6 sponsored slots that repeat organic TCINs (`is_sponsored_sku: true`). For "lego" pages returned 27 and 24 products, and one TCIN appeared on both pages (47 unique from 2 pages, total_results 662). The counts are not stable between runs: another "air fryer" run on the same day got 24, 47 and 71 unique because one organic TCIN repeated across pages, and `total_pages` for the same query came back as 12 on one call and 11 on another. The block drops sponsored copies and dedupes by TCIN, so treat the counts as approximate and stop on an empty page.

### 2. Product details by TCIN (plain HTTP, no browser)

```python
import html, json, re

TCIN = "89110863"   # from the search rows above, or the A-<digits> part of any /p/ URL

page = http_get(f"https://www.target.com/p/-/A-{TCIN}", headers={
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 "
                  "(KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36",
    "Accept": "text/html,application/xhtml+xml",
    "Accept-Language": "en-US,en;q=0.9",
})
m = re.search(r'<script id="__NEXT_DATA__"[^>]*>(.*?)</script>', page, re.S)
if not m:
    raise SystemExit("no __NEXT_DATA__ (block page or layout change)")
queries = (json.loads(m.group(1)).get("props") or {}).get("dehydratedState", {}).get("queries") or []
layout = next((q["state"]["data"]["data"] for q in queries
               if str(q.get("queryKey", [""])[0]).endswith("fetch-cdui-layout-v1")), None)
core = next((mod["module_data"]["data"]["product"] for mod in (layout or {}).get("data_source_modules") or []
             if mod.get("module_type") == "ProductDetailWebDatasourceCore"), None)
if not core:
    raise SystemExit("no ProductDetailWebDatasourceCore module (block page, bad TCIN or layout change)")

item = core.get("item") or {}
desc = item.get("product_description") or {}
stats = (core.get("ratings_and_reviews") or {}).get("statistics") or {}
rating = stats.get("rating") or {}
images = (item.get("enrichment") or {}).get("image_info") or {}
detail = {
    "tcin": core.get("tcin"),
    "title": html.unescape(desc.get("title") or ""),
    "brand": (item.get("primary_brand") or {}).get("name"),
    "dpci": item.get("dpci"),
    "upc": (item.get("primary_barcode") or None),
    "url": (item.get("enrichment") or {}).get("buy_url"),
    "category_path": [b["name"] for b in (core.get("category") or {}).get("breadcrumbs") or []][1:],
    "rating": rating.get("average"),
    "rating_count": rating.get("count"),
    "review_count": stats.get("review_count"),
    "question_count": stats.get("question_count"),
    "rating_distribution": rating.get("distribution"),
    "highlights": [html.unescape(re.sub("<[^>]+>", "", b)) for b in (desc.get("soft_bullets") or {}).get("bullets") or []],
    "specs": [html.unescape(re.sub("<[^>]+>", "", b)) for b in desc.get("bullet_descriptions") or []],
    "description": html.unescape(re.sub("<[^>]+>", " ", desc.get("downstream_description") or "")).strip(),
    "package_dimensions": item.get("package_dimensions"),
    "primary_image": (images.get("primary_image") or {}).get("url"),
    "image_count": 1 + len(images.get("alternate_images") or []),
}
print(json.dumps(detail, indent=1)[:2500])
```

Observed output (trimmed):

```
{
 "tcin": "89110863",
 "title": "Gourmia 6qt Digital Window Air Fryer with 12 Presets & Guided Cooking Black: Stainless Steel, 1-Year Warranty",
 "brand": "Gourmia",
 "dpci": "072-04-0231",
 "upc": "810002864582",
 "category_path": ["Kitchen & Dining", "Kitchen Appliances", "Fryers", "Air Fryers"],
 "rating": 4.07,
 "rating_count": 2296,
 "review_count": 1017,
 "question_count": 242,
 "rating_distribution": {"rating1": 379, "rating2": 108, "rating3": 73, "rating4": 153, "rating5": 1583},
 "highlights": ["6-QT capacity can cook family staples and party favorites for a variety of crowds", ...],
 "specs": ["Dimensions (Overall): 13.58 Inches (H) x 14.05 Inches (W) x 11.46 Inches (D)", "Weight: 12.1 Pounds", ...],
 "description": "Check on your food at-a-glance with the Gourmia 6-QT. Digital Window Air Fryer. ..."
}
```

The same request from plain Python `urllib` (desktop Chrome UA) also returned 200 and a 360 KB page. An unknown TCIN (`A-1`) returned HTTP 404, which `http_get` raises as `HTTPError`.

### 3. Live price and fulfillment on a product page (browser)

```python
import json, time

TCIN = "89110863"

tab = new_tab(f"https://www.target.com/p/-/A-{TCIN}")
wait_for_load()
# The price module stays a grey placeholder while the tab is hidden
# (document.visibilityState == "hidden"), so bring this tab to the front.
activate_tab(current_tab())

# Price and fulfillment are deferred modules, filled after load by
# www.target.com/cdui_orchestrations/v1/pages/pdp/deferred_enrichment/modules
live = {}
for _ in range(20):
    live = js("""(() => {
      const t = s => (document.querySelector(s)?.innerText || '').trim();
      return {
        visibility: document.visibilityState,
        title: t('[data-test="product-title"]'),
        price_block: t('[data-test="module-product-detail-price-v2"]'),
        fulfillment: t('[data-test="module-product-detail-fulfillment-v1"]'),
        rating_block: t('[data-test="ratingFeedbackContainer"]'),
      };
    })()""")
    if live["price_block"] and live["fulfillment"]:
        break
    time.sleep(1)
print(json.dumps(live, indent=1))
close_tab(tab)
```

Observed output (store name trimmed):

```
{
 "visibility": "visible",
 "title": "Gourmia 6qt Digital Window Air Fryer with 12 Presets & Guided Cooking Black: Stainless Steel, 1-Year Warranty",
 "price_block": "$59.99 reg $69.99\n$10 off",
 "fulfillment": "Pickup\nReady within 2 hours\nDelivery\nAs soon as 3pm today\nShipping\nArrives by Sun, Oct 25\nPickup at <nearest store>\nOnly 1 left\nAlways free\nChange",
 "rating_block": "4.07 out of 5 stars\n2,296\n242 Questions"
}
```

If you only need price and stock for a TCIN, the search rows from block 1 already carry them, so this step is optional.

## Fields

Search (`search_response.products[i]`, block 1):

| Field | Path | Example / notes |
|---|---|---|
| TCIN | `tcin` | `"89110863"`, Target's item id, the `A-<digits>` in URLs |
| Title | `item.product_description.title` | HTML-escaped (`&#38;`), run `html.unescape` |
| Brand | `item.primary_brand.name` | `"Gourmia"` |
| Price (display) | `price.formatted_current_price` | `"$59.99"`, can be `"See price in cart"` (then `price.hide_price == "SEE_PRICE_IN_CART"`) |
| Price (number) | `price.current_retail`, `price.reg_retail` | `59.99`, `69.99`; `parent.price` on variation items has only the formatted strings |
| Rating | `ratings_and_reviews.statistics.rating.average` | `4.07`; for variation items it lives on `parent.ratings_and_reviews` |
| Rating count | `ratings_and_reviews.statistics.rating.count` | `2296` (star ratings; written reviews are fewer, see PDP `review_count`) |
| URL | `item.enrichment.buy_url` | absolute `https://www.target.com/p/<slug>/-/A-<tcin>` |
| Shipping | `fulfillment.shipping_options.availability_status` | only `IN_STOCK` appeared in 119 rows checked; treat anything else as not shippable |
| Pickup | `fulfillment.store_options[0].order_pickup.availability_status` | `IN_STOCK`, `OUT_OF_STOCK`, `UNAVAILABLE` (with `reason_code`) |
| Sold out | `fulfillment.sold_out`, `fulfillment.is_out_of_stock_in_all_store_locations` | booleans |
| Sponsored | `is_sponsored_sku` | `true` on ad slots |
| Images | `item.enrichment.image_info.primary_image.url` | scene7 URL |
| Paging | `search_response.metadata` | `current_page`, `total_pages`, `total_results`, `count` (24), `offset`, `keyword` |

Product page (`ProductDetailWebDatasourceCore.module_data.data.product`, block 2): `tcin`, `item.product_description.{title,bullet_descriptions,soft_bullets.bullets,downstream_description}`, `item.dpci`, `item.primary_barcode` (UPC), `item.primary_brand`, `item.package_dimensions`, `item.enrichment.image_info`, `category.breadcrumbs`, `ratings_and_reviews.statistics.{rating.average,rating.count,rating.distribution,review_count,question_count,recommended_percentage}`.

## Gotchas

- **Do not rebuild the slp URL from scratch.** Without `sapphire_channel` it returned 400 `Required argument [SapphireChannel sapphire_channel] not specified`; with the full parameter set from plain Python it returned 435 (PerimeterX). Copy the URL the page sent (block 1) and only change `offset` and `query_string`.
- **Page size is fixed at 24.** `count=48` and `count=96` returned HTTP 206 with no `search_response`. Keep `count=24` and walk `offset` in steps of 24.
- **Depth cap.** `offset` above 1199 returned 400 `getSlp.request.offset: must be less than or equal to 1199`, so one query yields at most about 1,200 results. An offset past the last page (e.g. page 29 of 28 for "lego") returned 200 with 0 products. Split broad searches by category or facet to go deeper.
- **Results depend on location.** The store, pickup status and some prices come from the store Target picks for the browser's IP (`store_id`, `zip` in the slp URL). Different exit IPs give different `pickup_status`.
- **Sponsored duplicates.** With `count=24`, "air fryer" pages returned 30 products; the 6 extras were sponsored repeats of organic TCINs. Product counts per page vary (27 and 24 for "lego"), and organic TCINs sometimes repeat on the next page, so dedupe by TCIN and loop until `metadata.total_pages` or an empty page, not on a fixed size. `total_pages` itself drifted (12 vs 11) between calls for the same query.
- **Variation items.** Items with a `parent` (colors/sizes) had no product-level `ratings_and_reviews`; read it from `parent`.
- **Query rewriting.** "lego" came back with `facets: "d_brand_all:lego"` (brand facet applied). A nonsense query (`zzqqxxnonexistent123`) still returned a 9-page grid, so check `metadata.keyword` and relevance before trusting results.
- **Hidden tabs do not render the PDP price.** In a background tab the price module stayed a placeholder (`data-test="price-module-placeholder"`) and `visibilityState` was `hidden`; `activate_tab(current_tab())` fixed it.
- **First-load PerimeterX block.** A block can happen on the first search load of a fresh browser, with no earlier requests. Signs: `performance.getEntriesByType('resource')` shows the slp entries with `responseStatus` 435, refetching the same URL also returns 435 (`<meta name="description" content="px-captcha">`), there are 0 `[data-test="ListingPageProductListing"]` cards, and the page says "We couldn't find a match for your search." That text is not a real empty result. Starting a new browser session with a different exit IP worked. Do not try to solve the captcha.
- **Browser fetch volume.** A full slp response is about 850 KB of JSON. In unblocked sessions, about 20 slp requests (page loads plus in-page fetches at 2 s spacing) produced no 429 and no new block; heavier rates were not tested, so keep the pacing.
- **Tabs.** Blocks 1 and 3 each open a tab with `new_tab()` and close it with `close_tab(tab)` at the end. Closing the last tab left the cloud browser with 0 tabs, and the next `new_tab()` still worked.
- **DOM selectors (fallback).** Search cards were `[data-test="ListingPageProductListing"]` (24 to 30 per page on desktop) and the pager `[data-test="listing-page-pagination"]` read `page 1 of 12`. Prefer the JSON, the cards are lazily filled.
- No cookie/consent banner blocked anything on www.target.com from a US exit IP.

## What does not work

- **Plain HTTP search**: `https://www.target.com/s?searchTerm=...` returned 200 but a 140 KB shell with zero `/p/` links; products are not server-rendered.
- **Plain HTTP API calls**: `cdui-orchestrations.target.com/.../pages/slp` and `redsky.target.com/redsky_aggregations/v1/web/pdp_client_v1` both returned **435** with a PerimeterX captcha payload. Use the browser instead of trying to get around it.
- **Search in a PerimeterX-flagged browser session**: when the first search load already got slp 435, the in-page `fetch()` of the same URL also returned 435. A new browser session (new exit IP) worked.
- **Price/stock from product HTML**: the `ProductDetailPrice` and fulfillment modules in `__NEXT_DATA__` are `is_enrichment_deferred: true` with no data; use block 1 or block 3.
- **JSON-LD**: none for products on search or product pages.
- **Search data in `__NEXT_DATA__`**: the search page's `__NEXT_DATA__` held only page content, nearby stores and global content, no products.
