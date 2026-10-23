# Wayfair — Search, Category and Product Scraping

Field-tested against www.wayfair.com on 2026-10-23 using a Browser Use cloud browser (search, categories, product pages) and plain urllib/curl (listing pages, plus a few product-page checks).

## Quick summary

- **Search and category grids**: every product card carries a JSON blob in the `data-tracking-metadata` attribute with SKU (`displayListingId`), name, brand, current price (`leadPrice`), was price (`strikethroughPrice`), average rating, review count, sponsored flag, option count and stock message. Reading that attribute is more stable than scraping the visible price/rating markup.
- **48 grid products per page.** Pagination is `&curpage=N` (or `?curpage=N` on a category URL); the page's own "Next" link (`a[data-click-location="PAGINATION:NEXT"]`) already has the right URL, including after a search redirect.
- **Searches often redirect to a category.** `keyword.php?keyword=coffee+table` landed on `/furniture/sb0/coffee-tables-c414602.html?redir=coffee+table&rtype=9`; `area rug 8x10` landed on the 8x10 rugs category. Long-tail queries (`boucle swivel accent chair`) stayed on `keyword.php`. Both page types use the same card markup.
- **Product pages have no Product JSON-LD** (only `BreadcrumbList` and `WebSite`). Details come from stable `data-rtl-id`, `data-test-id` and `data-name` hooks: title, brand, SKU, price, was price, rating, review count, stock, selected option, description, feature bullets, specification table, dimensions, breadcrumbs, gallery images.
- **Plain HTTP does not work reliably.** The listing HTML embeds the same card metadata, but PerimeterX answered 19 of 31 plain listing requests and 3 of 3 product-page requests with HTTP 429 today, whatever the User-Agent (details under "What does not work"). Both working blocks use the browser.
- No login, consent wall or captcha was shown in the cloud browser across 23 page loads (two separate sessions) with 2-3 s pauses.

## What works

### 1. Search or category, several pages (browser)

```python
import json, re, time, urllib.parse

QUERY = "coffee table"   # free text; Wayfair may redirect it to a category page
START_URL = None         # or a category URL, e.g. "https://www.wayfair.com/furniture/sb0/coffee-tables-c414602.html"
MAX_PAGES = 3            # explicit safety cap; the run says so when it stops because of it
PAUSE = 3                # seconds between page loads
GRID_SIZE = 48           # grid cards on every full page seen today

def wait_new_document(check, timeout=45):
    # Wait for a NEW wayfair.com document (old one is tagged with window.__bhPrev) to finish loading,
    # so we never read the previous page's readyState or cards.
    deadline, last = time.time() + timeout, None
    while time.time() < deadline:
        try:
            last = js("[location.href, document.readyState, !!window.__bhPrev]")
        except (TimeoutError, RuntimeError):   # context destroyed or eval timeout while navigating
            last = None
        if last and last[0].startswith("https://www.wayfair.com/") and last[1] == "complete" and not last[2]:
            if not check(last[0]):
                raise RuntimeError(f"landed on an unexpected page: {last[0]}")
            return last[0]
        time.sleep(0.5)
    raise TimeoutError(f"page did not finish loading in {timeout}s; last state {last}")

def open_url(url, check=lambda u: True, attempts=3):
    for i in range(attempts):
        try:
            try:
                here = js("location.href") or ""
            except (TimeoutError, RuntimeError):
                here = ""
            if here.startswith("https://www.wayfair.com/"):
                js("window.__bhPrev = true")
                goto_url(url)
            else:
                new_tab(url)   # opens in the background; this skill never brings Chrome to the front
            return wait_new_document(check)
        except TimeoutError as e:   # browser-call and load timeouts are retried, then raised
            if i == attempts - 1:
                raise
            print(f"timeout opening {url}: {e}; retry {i + 1}")
            time.sleep(5 * (i + 1))

# Grid cards have listingCardName; sponsored-carousel cards at the top use another shape and are skipped.
CARDS_JS = r"""
[...document.querySelectorAll('[data-test-id="ListingCard"]')].map(card => {
  let m = {};
  try { m = JSON.parse(card.querySelector('[data-tracking-metadata]').getAttribute('data-tracking-metadata')).metadata || {}; } catch (e) {}
  if (!m.listingCardName) return null;
  const a = card.querySelector('a[href*="/pdp/"]');
  return {sku: m.displayListingId ?? null, name: m.listingCardName, brand: m.manufacturerName ?? null,
          price: m.leadPrice ?? null, was_price: m.strikethroughPrice ?? null,
          rating: m.averageRating ?? null, review_count: m.totalReviewCount ?? null,
          sponsored: m.isSponsored === true, options: m.choicesText ?? null,
          stock: m.inventoryStatusMessage ?? null, href: a ? a.href : null};
}).filter(Boolean)
"""
NEXT_JS = "(() => { const a = document.querySelector('a[data-click-location=\"PAGINATION:NEXT\"]'); return a ? a.href : null; })()"

def text(v):
    if isinstance(v, (int, float)) and not isinstance(v, bool):
        return str(v)
    return v.strip() or None if isinstance(v, str) else None

def number(v):
    if isinstance(v, bool):
        return None
    if isinstance(v, (int, float)):
        return v
    m = re.search(r"\d[\d,]*(?:\.\d+)?", v) if isinstance(v, str) else None
    return float(m.group().replace(",", "")) if m else None

def clean_url(href):
    # keep the product path + piid (selected variant); drop ad/tracking params
    if not isinstance(href, str) or not href:
        return None
    u = urllib.parse.urlsplit(href)
    piid = urllib.parse.parse_qs(u.query).get("piid", [None])[0]
    return f"https://www.wayfair.com{u.path}" + ("?" + urllib.parse.urlencode({"piid": piid}) if piid else "")

def read_cards(timeout=30):
    # Cards hydrate a few seconds after load. Done = a full grid, or (last page only) a count that
    # stopped changing with no Next link. Anything else raises instead of returning a short page.
    deadline, prev, rows, nxt = time.time() + timeout, None, [], None
    while time.time() < deadline:
        try:
            rows, nxt = js(CARDS_JS) or [], js(NEXT_JS)
        except (TimeoutError, RuntimeError) as e:
            print("card read failed, retrying:", e)
            time.sleep(1.5)
            continue
        if len(rows) >= GRID_SIZE or (rows and not nxt and len(rows) == prev):
            return rows, nxt
        prev = len(rows)
        time.sleep(1.5)
    raise RuntimeError(f"only {len(rows)} product cards after {timeout}s (next link: {bool(nxt)}) at {page_info()}")

def curpage_is(n):
    return lambda u: urllib.parse.parse_qs(urllib.parse.urlsplit(u).query).get("curpage") == [str(n)]

url = START_URL or "https://www.wayfair.com/keyword.php?" + urllib.parse.urlencode({"keyword": QUERY})
results, seen = [], set()
for page in range(1, MAX_PAGES + 1):
    landed = open_url(url, check=(lambda u: True) if page == 1 else curpage_is(page))
    rows, url = read_cards()
    new = 0
    for r in rows:
        r["sku"] = text(r["sku"])
        if r["sku"] is not None:          # dedupe by SKU (sponsored items can repeat); keep rows without one
            if r["sku"] in seen:
                continue
            seen.add(r["sku"])
        new += 1
        for k in ("name", "brand", "price", "was_price", "options", "stock"):
            r[k] = text(r[k])
        r["rating"] = number(r["rating"])
        rc = number(r["review_count"])
        r["review_count"] = int(rc) if rc is not None else None
        r["url"] = clean_url(r.pop("href"))
        r["price_num"], r["was_price_num"] = number(r["price"]), number(r["was_price"])
        r["on_sale"] = r["was_price"] is not None
        r["page"] = page
        results.append(r)
    print(f"page {page}: {len(rows)} cards, {new} new, url={landed}")
    if not url:
        print("last page reached (no Next link)")
        break
    time.sleep(PAUSE)
else:
    print(f"stopped at the MAX_PAGES={MAX_PAGES} cap; more pages exist, next={url}")

print(len(results), "products")
print(json.dumps(results[:1], indent=1))
```

Observed output (trimmed):

```
page 1: 48 cards, 48 new, url=https://www.wayfair.com/furniture/sb0/coffee-tables-c414602.html?redir=coffee+table&rtype=9
page 2: 48 cards, 48 new, url=https://www.wayfair.com/furniture/sb0/coffee-tables-c414602.html?redir=coffee+table&rtype=9&curpage=2
page 3: 48 cards, 48 new, url=https://www.wayfair.com/furniture/sb0/coffee-tables-c414602.html?redir=coffee+table&rtype=9&curpage=3
stopped at the MAX_PAGES=3 cap; more pages exist, next=https://www.wayfair.com/furniture/sb0/coffee-tables-c414602.html?redir=coffee+table&rtype=9&curpage=4
144 products
[
 {
  "sku": "W113456401",
  "name": "Coffee Table with Storage for Living Room",
  "brand": "George Oliver",
  "price": "$106.99",
  "was_price": "$148.99",
  "rating": 4.6,
  "review_count": 598,
  "sponsored": true,
  "options": "2 Colors",
  "stock": "815 Left in Stock",
  "url": "https://www.wayfair.com/furniture/pdp/george-oliver-coffee-table-with-storage-for-living-room-w113456401.html?piid=1077057334",
  "price_num": 106.99,
  "was_price_num": 148.99,
  "on_sale": true,
  "page": 1
 }
]
```

Set `START_URL` to any category URL to browse instead of search (the area rugs category `https://www.wayfair.com/rugs/sb0/area-rugs-c215386.html` returned 48 cards per page the same way). Sponsored products sit inside the grid with `sponsored: true`; filter them out if you only want organic ranking. A page that does not reach 48 cards within 30 s while a Next link exists raises instead of returning a short page.

### 2. Product page details (browser)

```python
import json, re, time, urllib.parse

URLS = [  # product URLs, e.g. the "url" values from the search block
    "https://www.wayfair.com/furniture/pdp/gracie-oaks-torron-lift-top-coffee-table-with-storage-w100225412.html",
    "https://www.wayfair.com/furniture/pdp/trent-austin-design-rundell-pomona-rustic-industrial-modern-vintage-metal-and-solid-wood-coffee-table-with-storage-easy-assembly-small-rectangle-coffee-table-42-in-x-24-in-x-18-in-w004213194.html?piid=1743852479",
]
PAUSE = 3

def wait_new_document(check, timeout=45):
    # Wait for a NEW wayfair.com document (old one is tagged with window.__bhPrev) to finish loading.
    deadline, last = time.time() + timeout, None
    while time.time() < deadline:
        try:
            last = js("[location.href, document.readyState, !!window.__bhPrev]")
        except (TimeoutError, RuntimeError):
            last = None
        if last and last[0].startswith("https://www.wayfair.com/") and last[1] == "complete" and not last[2]:
            if not check(last[0]):
                raise RuntimeError(f"landed on an unexpected page: {last[0]}")
            return last[0]
        time.sleep(0.5)
    raise TimeoutError(f"page did not finish loading in {timeout}s; last state {last}")

def open_url(url, check=lambda u: True, attempts=3):
    for i in range(attempts):
        try:
            try:
                here = js("location.href") or ""
            except (TimeoutError, RuntimeError):
                here = ""
            if here.startswith("https://www.wayfair.com/"):
                js("window.__bhPrev = true")
                goto_url(url)
            else:
                new_tab(url)   # background tab
            return wait_new_document(check)
        except TimeoutError as e:
            if i == attempts - 1:
                raise
            print(f"timeout opening {url}: {e}; retry {i + 1}")
            time.sleep(5 * (i + 1))

PDP_JS = r"""
(() => {
  const q = s => document.querySelector(s);
  const txt = s => { const e = q(s); return e ? e.innerText.trim() : null; };
  const dl = root => { const out = {}; if (!root) return out;
    root.querySelectorAll('dt').forEach(dt => { const dd = dt.nextElementSibling;
      if (dd && dd.tagName === 'DD') out[dt.textContent.trim()] = dd.textContent.trim().replace(/\s+/g, ' '); });
    return out; };
  let crumbs = [];
  document.querySelectorAll('script[type="application/ld+json"]').forEach(s => {
    let d; try { d = JSON.parse(s.text); } catch (e) { return; }
    const nodes = Array.isArray(d) ? d : (d && d['@graph']) || [d];
    nodes.forEach(n => {
      if (!n || n['@type'] !== 'BreadcrumbList' || !Array.isArray(n.itemListElement)) return;
      crumbs = n.itemListElement.map(i => i.name || (i.item && typeof i.item === 'object' ? i.item.name : null)).filter(Boolean);
    });
  });
  const skuCrumb = [...document.querySelectorAll('[data-test-id^="breadcrumbs-crumb-"]')]
    .map(e => e.innerText.trim()).find(t => /^SKU:/i.test(t)) || null;
  const imgs = [...document.querySelectorAll('[data-name="MediaTray"] img')]
    .filter(i => i.alt !== 'Wayfair Verified').map(i => i.currentSrc || i.src).filter(Boolean);
  return {
    title: txt('[data-rtl-id="listingHeaderNameHeading"]'),
    brand: txt('[data-rtl-id="listingManufacturerName"]'),
    sku_crumb: skuCrumb,
    price: txt('[data-test-id="PricingFull-leadPrice"]'),
    was_price: txt('[data-test-id="PricingFull-strikethroughPrice"] s'),
    rating: txt('[data-rtl-id="reviewsRatingReviewAverage"]'),
    reviews: txt('[data-rtl-id="reviewsRatingReviewCount"]'),
    stock: txt('[data-test-id="InventoryWidgetDisplay-Text"]'),
    delivery: txt('[data-test-id="DeliveryPromise"]'),
    selected_option: txt('[data-name="Features_Choices"]'),
    description: txt('[data-name="RomanceCopy"]'),
    features: txt('[data-name="FeatureBullets"]'),
    specs: dl(q('[data-name="SpecificationsDetails"]')),
    dimensions: dl(q('[data-name="StructuredDimensions"]')),
    breadcrumbs: crumbs,
    canonical: (q('link[rel="canonical"]') || {}).href || null,
    images: imgs,
  };
})()
"""

def strip_label(s, label):
    return re.sub(rf"^\s*{label}\s*", "", s).strip() if isinstance(s, str) else None

def sku_token(url):
    m = re.search(r"-([a-z0-9]+)\.html", urllib.parse.urlsplit(url).path.lower())
    return m.group(1) if m else None

def product_details(url):
    want = sku_token(url)
    landed = open_url(url, check=lambda u: want is None or f"-{want}.html" in urllib.parse.urlsplit(u).path.lower())
    d = {}
    for i in range(30):                      # wait for hydration: title + price, then the spec table
        try:
            d = js(PDP_JS) or {}
        except (TimeoutError, RuntimeError) as e:
            print("read failed, retrying:", e)
        if d.get("title") and d.get("price") and (d.get("specs") or i >= 20):
            break
        time.sleep(1)
    if not d.get("title"):
        raise RuntimeError(f"no product header after 30 s at {landed}")
    sku = (d.pop("sku_crumb") or "").split(":", 1)[-1].strip() or (want.upper() if want else None)
    # gallery thumbnails are tiny (h72-w72 seen); the resize segment is not signed, so ask for 1200px
    big, seen = [], set()
    for src in d.pop("images") or []:
        m = re.search(r"/(\d{6,})/[^/]+\.jpg", src)
        key = m.group(1) if m else src     # dedupe by image id; keep images whose id we cannot parse
        if key in seen:
            continue
        seen.add(key)
        big.append(re.sub(r"/resize-[^/]*?(%5E|\^)", r"/resize-h1200-w1200\1", src))
    rating = re.search(r"\d+(?:\.\d+)?", d.get("rating") or "")
    cnt = re.search(r"\d[\d,]*", d.get("reviews") or "")
    out = {
        "sku": sku,
        "url": landed,
        **d,
        "brand": strip_label(d.get("brand"), "By"),
        "selected_option": " ".join((d.get("selected_option") or "").split()) or None,
        "description": strip_label(d.get("description"), "About This Product"),
        "features": [f.strip() for f in (strip_label(d.get("features"), "Features") or "").split("\n") if f.strip()],
        "rating": float(rating.group()) if rating else None,
        "reviews": int(cnt.group().replace(",", "")) if cnt else None,
        "images": big,
    }
    # fields that came back empty, so a partial read is visible instead of silent
    out["missing"] = [k for k in ("price", "rating", "reviews", "description", "features", "specs",
                                  "dimensions", "breadcrumbs", "images") if not out.get(k)]
    return out

out, failed = [], []
for i, u in enumerate(URLS):
    if i:
        time.sleep(PAUSE)
    try:
        out.append(product_details(u))
    except (TimeoutError, RuntimeError) as e:
        failed.append((u, str(e)))
for p in out:  # trimmed preview
    print(json.dumps({**p, "description": (p.get("description") or "")[:120],
                      "images": (p.get("images") or [])[:2], "image_count": len(p.get("images") or [])}, indent=1))
if failed:
    raise RuntimeError(f"{len(failed)} of {len(URLS)} product pages failed: {failed}")
```

Observed output (first product, trimmed):

```
{
 "sku": "W100225412",
 "url": "https://www.wayfair.com/furniture/pdp/gracie-oaks-torron-lift-top-coffee-table-with-storage-w100225412.html?piid=1032580731",
 "title": "Torron Lift Top Coffee Table with Storage",
 "brand": "Gracie Oaks",
 "price": "$204.99",
 "was_price": "$499.00",
 "rating": 4.5,
 "reviews": 2479,
 "stock": "6 Left in Stock",
 "delivery": "FREE Fast Delivery",
 "selected_option": "Color: Rustic Brown",
 "description": "This wooden coffee table features a versatile design and four unique ways to meet your daily needs, making it the perfec",
 "features": [
  "A unique double pneumatic lift top design, effortlessly converting from a coffee table to dining table or workstation",
  "A hidden storage compartment to tuck away odds and ends, plus a lower shelf for a variety of items",
  "Crafted from wood with a distinctive finish of your choice"
 ],
 "specs": {
  "Product Type": "Coffee Table",
  "Top Color": "Rustic Brown",
  "Top Material": "Manufactured Wood",
  ...
  "Weight Capacity": "110 lb.",
  ...
  "Overall Shape": "Rectangular",
  ...
 },
 "dimensions": {
  "Extended": "25'' H X 42'' W X 39'' D",
  "Overall Dimensions": "20'' H X 42'' W X 20'' D",
  "Overall Product Weight": "68 lb."
 },
 "breadcrumbs": ["Furniture", "Living Room Furniture", "Coffee Tables & End Tables", "Coffee Tables"],
 "canonical": "https://www.wayfair.com/furniture/pdp/gracie-oaks-w100225412.html",
 "images": [
  "https://assets.wfcdn.com/im/41698685/resize-h1200-w1200%5Ecompr-r85/2891/289195857/Torron+Lift+Top+Coffee+Table+with+Storage-1032580731.jpg",
  ...
 ],
 "missing": [],
 "image_count": 10
}
```

The second product (Pomona, W004213194) returned `"stock": null` because that page had no inventory badge, `"missing": []` and 11 images. A URL without `piid` came back with the default variant's `piid` added to the address in one session and without it in another. A page that loads something other than the requested SKU (redirect) raises instead of returning that other product; failures are collected and raised after the preview print.

## Fields

Block 1, one row per grid card (card values come from `data-tracking-metadata` -> `metadata`):

| Output field | Source | Example | Notes |
|---|---|---|---|
| `sku` | `displayListingId` | `W113456401` | Wayfair SKU. Newer ones are `W` + 9 digits; older ones are letters + digits (`QSI12828`). Also shown on the product page as "SKU: ..." |
| `name` | `listingCardName` | `Coffee Table with Storage for Living Room` | |
| `brand` | `manufacturerName` | `George Oliver` | |
| `price` | `leadPrice` | `$106.99` | Current price string. When `was_price` is set this is the sale price |
| `was_price` | `strikethroughPrice` | `$148.99` | `null` when the item is not discounted |
| `rating` | `averageRating` | `4.6` | Number; up to 2 decimals on cards, 1 decimal on the product page |
| `review_count` | `totalReviewCount` | `598` | Integer |
| `sponsored` | `isSponsored` | `true` | Paid placement inside the grid |
| `options` | `choicesText` | `2 Colors` | |
| `stock` | `inventoryStatusMessage` | `815 Left in Stock` | Often `null` |
| `url` | card `a[href*="/pdp/"]` | `.../pdp/...-w113456401.html?piid=1077057334` | `piid` = the variant shown on the card; its price is the card price |
| `price_num` | parsed `price` | `106.99` | |
| `was_price_num` | parsed `was_price` | `148.99` | `null` when not discounted |
| `on_sale` | `was_price` present | `true` | |
| `page` | loop counter | `1` | Result page the row came from |

Other keys present in the metadata but not returned: `shippingBadgeText` ("FREE 3-Day Delivery"), `deliveryEstimateText` ("Get it by Mon, Oct 26"), `flagText` ("10% Off With Code in App"), `imageId`, `displayedOptionIDs`.

Block 2, one object per product page: `sku`, `url` (address after load), `title`, `brand`, `price`, `was_price`, `rating` (number), `reviews` (integer), `stock`, `delivery`, `selected_option`, `description`, `features` (list), `specs` (dict), `dimensions` (dict), `breadcrumbs` (list), `canonical`, `images` (list of 1200px URLs), `missing` (list of names of core fields that came back empty; `[]` when complete). The preview print also adds `image_count`.

## Gotchas

- **Hydration delay.** One search page had 30 of 48 cards readable at 1-2 s after load and all 48 at about 5 s. Block 1 polls until the grid is full; a single read right after the load can come back short.
- **Top carousel is not the grid.** Some pages start with a sponsored-shop carousel of `ListingCard`s whose metadata uses a different shape (`firstPriceValue`, no `listingCardName`) and repeats a few products. Page 1 of the coffee-table category had 12 such cards in one check and 14 in another; the 8x10 rug page had 14; long-tail `keyword.php` pages had 2. Filtering on `listingCardName` left exactly the 48 grid cards every time.
- **Search redirects.** Short head terms redirect to a category URL with `redir=` and `rtype=9`. Read pagination from the page's Next link instead of building `keyword.php?...&curpage=N` yourself.
- **Page 200 is where the Next links stop, not where results stop.** Following Next links ended at page 200 in every check, even for a narrow query (`walnut nightstand with charging station` still returned 48 cards on page 200, with no Next link). A hand-built `&curpage=201` on the same query still returned 48 cards and a Next link to 202. Results that deep are loosely related; set `MAX_PAGES` and filter.
- **Sponsored repeats.** A product sponsored on page 1 can show up again organically on a later page. Block 1 dedupes by SKU and keeps rows that have none.
- **Location-dependent fields.** Delivery estimates and the "to <ZIP>" line in the shipping widget follow the browser's location; the cloud browser was assigned one automatically. Do not treat delivery dates as universal; price changes by location were not tested.
- **Variant prices.** A product URL without `piid` shows the default variant; with `piid` it shows that color/size. Rugs list a price for the shown size only. Cards on rug pages also show an "Open Box" price in the visible text that is not in the metadata.
- **Image URLs.** The `/resize-hNNN-wNNN^.../` segment in `assets.wfcdn.com/im/...` URLs is not signed: the gallery thumbnail (`resize-h72-w72`) was 2-3 KB and the same URL with `resize-h1200-w1200` returned a full-size JPEG (52-492 KB across checks). Exclude the "Wayfair Verified" stamp images (alt text) from the gallery.
- **Canonical link is shortened** (`/furniture/pdp/gracie-oaks-w100225412.html`); the slug URL and the canonical both identify the same SKU.
- **Selectors.** The attributes used here (`data-test-id`, `data-rtl-id`, `data-name`, `data-tracking-metadata`) are testing/analytics hooks, steadier than the hashed CSS class names (`_16fu6t1...`), which change between builds. If a block raises "no product cards" or `missing` fills up, re-check these names first.
- **Pacing.** 2-3 s between page loads produced no blocks in the browser. Plain HTTP drew 429s even with 5-30 s spacing and 15-90 s back-off.
- **Background tabs.** `new_tab()` opens the tab in the background and everything here works without bringing Chrome to the front; no `activate_tab()` is needed.

## What does not work

- **Product JSON-LD**: none on product pages; only `BreadcrumbList` and `WebSite`. The coffee-table category page carried `BreadcrumbList`, `FAQPage` and `WebSite`, no `ItemList`.
- **Plain HTTP product pages**: `/pdp/` URLs returned HTTP 429 (PerimeterX) in 3 of 3 attempts from plain curl with a browser User-Agent. Use block 2.
- **Plain HTTP listing pages**: when a plain request gets HTTP 200, the HTML carries the same `data-tracking-metadata` attributes as block 1 (48 grid cards per page). But across 31 plain requests to category pages today (Python urllib and curl, 5-30 s apart, some with 15-90 s back-off), 12 returned the page and 19 returned HTTP 429 from PerimeterX, and it got worse through the day: the last 12 requests from the same machine got one 200. A truthful client User-Agent and a Chrome User-Agent were both blocked; the single late success was curl with a Chrome User-Agent. No `Retry-After` header was sent. Use block 1.
- **Total result count**: no "N results" text was found on search or category pages; only the pager.
- **Not tested**: reviews pagination beyond the 5 review cards shown on the product page, sorting and filter facets in the URL, and non-US Wayfair domains.
