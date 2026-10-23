# Home Depot — Product Search, Specs & Store Availability

Field-tested against www.homedepot.com on 2026-10-23 using a Browser Use cloud browser.

## Quick summary

- **Plain HTTP is blocked.** `curl https://www.homedepot.com/s/cordless%20drill` returned **HTTP 403** and a 2,259-byte "Error Page" ("Oops!! Something went wrong. Please refresh page") with curl's own User-Agent (`curl/8.7.1`) and with a desktop Chrome UA alike; `/p/320326875` was 403 too. There is no plain-HTTP path, so every block uses the browser (and its own UA).
- **Search** (`/s/<query>`) either stays a search page or redirects to a category page (`"cordless drill"` went to `/b/Tools-Power-Tools-Drills/Cordless/N-5yc1vZc27fZ1z140i3`). Both carry the product data as JSON, not just HTML:
  - server-rendered Apollo cache `window.__APOLLO_STATE__` (`SearchModel` -> `products(...)` -> `base-searchNav-<itemId>` entries), and
  - the page's own `POST https://apionline.homedepot.com/federation-gateway/graphql?opname=searchModel`.
  On `/s/lawn%20mower` the server rendered the full grid and no searchModel call was made. On the `/b/...` redirect the server rendered only 12 and the page fetched the full 24 via GraphQL. Block 1 handles both.
- **Encode the query fully.** `urllib.parse.quote("1/2 copper pipe")` keeps the `/`, and `/s/1/2%20copper%20pipe` searched for `1`. `quote(QUERY, safe="")` gives `/s/1%2F2%20copper%20pipe`, which searched for `1/2 copper pipe` (4051 results).
- **The redirect is client-side and can be slow.** In a fresh session, `wait_for_load()` returned while the tab was still on `/s/cordless%20drill`; the URL changed to `/b/...?NCNI-5&searchRedirect=cordless%20drill&...` about 12 s later. Block 1 polls until the URL is stable and the grid data is there before it reads the path.
- **Pagination** is `?Nao=<offset>` on the final path, 24 products per page. On a redirecting query, `/s/<query>?Nao=24` redirects again to the category page and drops the offset (startIndex 0, page 1 again), so add `?Nao=` to the `/b/...` path. Block 1 raises if a page comes back at a different path or `startIndex`, and prints why it stopped (short page, page of repeats, or the `MAX_PAGES` cap).
- **Store.** A brand-new session's first page was priced and stocked for store 121 (Cumberland) before the site's script set the `THD_LOCALIZER` cookie; later pages used a store near the exit IP. The blocks open the home page first and wait for that cookie, and block 1 tags every row with the store of its own response.
- **Product page** (`/p/<itemId>` redirects to `/p/<slug>/<itemId>`): JSON-LD `Product` (name, brand, model, sku, gtin13, price, rating, review count, images, 10 reviews) plus the page's `opname=productClientOnlyProduct` GraphQL call, which has the spec table and pickup/delivery inventory for the store the site picked from the browser's location.
- **Other stores**: re-sending that productClientOnlyProduct request with a different `storeId` returned that store's pickup inventory (block 3). The express-delivery row did not change with the requested `storeId`; it stayed what the session's own store gets.
- **Rate limit is real.** After about 30 page loads plus ~12 API calls in ~25 minutes, every URL in that browser session (search and product pages) returned the 403 "Error Page". A new browser session worked. Keep runs small and paced. The blocks retry an Error Page or a failed call twice with backoff and then raise; they never return a partial result as if it were complete.

## What works

### 1. Search, 2+ pages (browser, JSON from the page itself)

```python
import json, re, time, urllib.parse
from email.utils import parsedate_to_datetime

QUERY = "lawn mower"      # any search term ('1/2 copper pipe' works: '/' is %-encoded below)
MAX_PAGES = 3             # safety cap; the run prints when it stops because of it

# Record the page's own searchModel GraphQL responses before any page script runs.
HOOK = """
(() => {
  if (window.__gql) return;
  window.__gql = [];
  const orig = window.fetch;
  window.fetch = async function (input, init) {
    const url = String((input && input.url) || input);
    const rec = url.includes('/federation-gateway/graphql?opname=searchModel') ? {url} : null;
    if (rec) window.__gql.push(rec);
    let res;
    try { res = await orig.apply(this, arguments); } catch (e) { if (rec) rec.err = String(e); throw e; }
    if (rec) {
      rec.status = res.status; rec.retryAfter = res.headers.get('Retry-After');
      try { rec.resp = await res.clone().text(); } catch (e) { rec.err = String(e); }
    }
    return res;
  };
})();
"""

# Products come from the server-rendered Apollo cache (window.__APOLLO_STATE__) or, when the
# server rendered only part of the grid, from the page's own POST ...graphql?opname=searchModel.
READ = """(() => {
  const st = window.__APOLLO_STATE__ || {};
  const sm = Object.values(st).find(v => v && v.__typename === 'SearchModel');
  const key = sm && Object.keys(sm).find(k => k.startsWith('products'));
  const ssr = key && Array.isArray(sm[key]) ? sm[key].map(r => r && st[r.__ref]).filter(Boolean) : [];
  const recs = window.__gql || [];
  const g = recs.filter(r => r.status === 200 && r.resp)
    .map(r => { try { return JSON.parse(r.resp).data.searchModel; } catch (e) { return null; } }).filter(Boolean).pop();
  const meta = (g || sm || {}).metadata || {};
  return {stale: !!window.__stale, ready: document.readyState, href: location.href,
          path: location.origin + location.pathname, title: document.title,
          cookie: document.cookie.includes('THD_LOCALIZER='),
          report: (g || sm || {}).searchReport || null, ssr, gql: g ? g.products : null, stores: meta.stores || null,
          failed: recs.filter(r => r.err || (r.status && r.status !== 200))
                      .map(r => ({status: r.status || null, retryAfter: r.retryAfter || null, err: r.err || null}))};
})()"""

def peek(expr):
    """js() that returns None while a navigation swaps the document.
    The harness reports a CDP timeout from js() as RuntimeError('Runtime.evaluate timed out ...')
    and from cdp() as TimeoutError; both are retried by the poll loops below."""
    try:
        return js(expr)
    except TimeoutError:
        return None
    except RuntimeError as e:
        if "timed out" in str(e) or "context" in str(e).lower():   # also "Execution context was destroyed"
            return None
        raise

def navigate(url):
    peek("window.__stale = true")   # marks the old document; polls skip it until the new one loads
    try:
        goto_url(url)
    except TimeoutError:
        pass   # Page.navigate can outlast the harness's 5 s IPC timeout; the navigation still runs

def retry_after_seconds(value, default):
    """Retry-After is either delay-seconds or an HTTP-date."""
    if not value:
        return default
    try:
        return max(0.0, float(value))
    except ValueError:
        pass
    try:
        return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())
    except (TypeError, ValueError):
        return default

def num(v):
    if isinstance(v, bool):
        return None
    if isinstance(v, (int, float)):
        return v
    if isinstance(v, str):
        try:
            return float(v)
        except ValueError:
            return None
    return None

def obj(v):
    return v if isinstance(v, dict) else {}

def arr(v):
    return v if isinstance(v, list) else []

def open_page(url, done, label, secs=45, tries=3):
    """Navigate, then poll READ on the new document until done(data) holds on two polls in a row
    (the URL is stable, so a client-side /s/ -> /b/ redirect has finished). The 403 'Error Page',
    a failed searchModel call or a timeout is retried with backoff, then raised."""
    for attempt in range(1, tries + 1):
        navigate(url)
        data, last, why = None, None, f"no complete product grid within {secs}s"
        for _ in range(secs):
            time.sleep(1)
            cur = peek(READ)
            if not cur or cur["stale"] or cur["ready"] != "complete":
                continue
            data = cur
            if "Error Page" in (data["title"] or ""):
                why = "403 'Error Page'"
                break
            if data["failed"] and not data["gql"]:
                f = data["failed"][-1]
                why = f"searchModel call failed (HTTP {f['status']} {f['err'] or ''})"
                break
            if done(data) and data["href"] == last:
                return data
            last = data["href"]
        if attempt == tries:
            raise RuntimeError(f"{label}: {why} on {tries} tries ({url}). Home Depot rate-limits browser "
                               "sessions: stop and continue later in a new browser session.")
        wait = 20 * attempt
        if data and data["failed"]:
            wait = retry_after_seconds(data["failed"][-1]["retryAfter"], wait)
        print(f"{label}: {why}; retry {attempt}/{tries - 1} in {wait:.0f}s")
        time.sleep(wait)

def grid_done(start):
    def done(d):
        rep = d["report"]
        if not isinstance(rep, dict):
            return False
        expected = min(24, max(int(num(rep.get("totalProducts")) or 0) - start, 0))
        return bool(d["gql"]) or len(d["ssr"]) >= expected
    return done

def field(o, name):
    # SSR keys carry arguments, e.g. 'pricing({"storeId":"121"})'; GraphQL keys do not.
    o = obj(o)
    if name in o:
        return o[name]
    return next((v for k, v in o.items() if k.startswith(name + "(")), None)

def stock(p):
    out = []
    for opt in arr(obj(field(p, "fulfillment")).get("fulfillmentOptions")):
        for svc in arr(obj(opt).get("services")):
            for loc in arr(obj(svc).get("locations")):
                inv = obj(obj(loc).get("inventory"))
                out.append({"service": svc.get("type"), "store": loc.get("storeName"), "store_id": loc.get("locationId"),
                            "qty": num(inv.get("quantity")), "in_stock": inv.get("isInStock"),
                            "limited": inv.get("isLimitedQuantity")})
    return out

def row(p, store_id):
    ids = obj(p.get("identifiers"))
    pricing = obj(field(p, "pricing"))
    rr = obj(obj(p.get("reviews")).get("ratingsReviews"))
    reviews = num(rr.get("totalReviews"))
    url = ids.get("canonicalUrl")
    item_id = p.get("itemId")
    return {
        "item_id": item_id if isinstance(item_id, str) and item_id else None,
        "title": ids.get("productLabel"),
        "brand": ids.get("brandName"),
        "model": ids.get("modelNumber"),
        "price": num(pricing.get("value")),
        "was_price": num(pricing.get("original")),
        "priced_for_store": store_id,          # metadata.stores.storeId of the same response
        "rating": num(rr.get("averageRating")),
        "reviews": int(reviews) if reviews is not None else None,
        "url": "https://www.homedepot.com" + url if isinstance(url, str) and url.startswith("/") else None,
        "sponsored": bool(obj(p.get("info")).get("isSponsored")),
        "stock": stock(p),
    }

tab = new_tab("about:blank")
cdp("Page.enable")
cdp("Page.addScriptToEvaluateOnNewDocument", source=HOOK)

# A brand-new session is served a default store until the site's own script sets the
# THD_LOCALIZER cookie. Open the home page first so every search page uses the same store.
open_page("https://www.homedepot.com/", lambda d: d["cookie"], "home page warm-up")
time.sleep(3)

data = open_page("https://www.homedepot.com/s/" + urllib.parse.quote(QUERY, safe=""), grid_done(0), "search page 1")
base = data["path"]   # /s/<query>, or the /b/<category> page the search redirected to
if not re.match(r"https://www\.homedepot\.com/(s|b)/", base):
    raise RuntimeError(f"search landed on an unexpected page: {data['href']}")
print("landed on:", base, "| keyword:", (data["report"] or {}).get("keyword"))

rows, seen, stores, stop = [], set(), {}, None
for page in range(MAX_PAGES):
    start = page * 24
    if page:
        time.sleep(4)   # polite pacing between page loads
        data = open_page(f"{base}?Nao={start}", grid_done(start), f"search page {page + 1}")
    rep = obj(data["report"])
    if data["path"] != base or num(rep.get("startIndex")) != start:
        # Seen: /s/<redirecting query>?Nao=24 bounced to the category page at startIndex 0.
        raise RuntimeError(f"page {page + 1}: got {data['href']} startIndex={rep.get('startIndex')}, "
                           f"expected {base} at {start}")
    products = [p for p in arr(data["gql"] or data["ssr"]) if isinstance(p, dict)]
    st = obj(data["stores"])
    stores[st.get("storeId")] = st
    ads = dupes = 0
    for p in products:
        r = row(p, st.get("storeId"))
        if r["sponsored"]:
            ads += 1
            continue
        if r["item_id"] is not None:
            if r["item_id"] in seen:
                dupes += 1
                continue
            seen.add(r["item_id"])
        rows.append(r)
    print(f"page {page + 1}: {len(products)} products from {'graphql' if data['gql'] else 'ssr'} "
          f"({ads} sponsored, {dupes} repeats), {len(rows)} unique so far, "
          f"store {st.get('storeId')}, totalProducts={rep.get('totalProducts')}")
    if len(products) < 24:
        stop = "end of results (short page)"
        break
    if len(products) == ads + dupes:
        stop = f"page {page + 1} added no new items; the rest of the {rep.get('totalProducts')} were NOT collected"
        break
else:
    stop = f"MAX_PAGES={MAX_PAGES} cap hit; about {rep.get('totalProducts')} products in total, the rest NOT collected"
print("stopped:", stop)

# The site never says "no match": a nonsense query still returns a grid of loosely related items.
words = [w for w in re.findall(r"[a-z0-9]+", QUERY.lower()) if len(w) > 1]
hits = sum(any(w in " ".join(str(r[k] or "") for k in ("title", "brand", "model")).lower() for w in words) for r in rows)
print(f"rows whose title/brand/model contain a query word: {hits}/{len(rows)}")
for sid, st in stores.items():
    print("store", sid, st.get("storeName"), "| nearby:",
          [(s.get("storeName"), s.get("storeId")) for s in arr(st.get("nearByStores")) if isinstance(s, dict)])
if len(stores) > 1:
    print("warning: pages were priced for different stores; use priced_for_store per row")
print(json.dumps(rows[:2], indent=1))
close_tab(tab)
```

Observed output (`QUERY = "lawn mower"`, trimmed):

```
landed on: https://www.homedepot.com/s/lawn%20mower | keyword: lawn mower
page 1: 26 products from ssr (2 sponsored, 0 repeats), 24 unique so far, store 910, totalProducts=2016
page 2: 26 products from ssr (2 sponsored, 4 repeats), 44 unique so far, store 910, totalProducts=2016
page 3: 24 products from ssr (2 sponsored, 6 repeats), 60 unique so far, store 910, totalProducts=2015
stopped: MAX_PAGES=3 cap hit; about 2015 products in total, the rest NOT collected
rows whose title/brand/model contain a query word: 60/60
store 910 Secaucus | nearby: [('North Bergen', '8475'), ('Holland Tunnel', '6845'), ('Manhattan West 23rd St', '6175'), ('Midtown Manhattan (Relo 6177)', '6140')]
[
 {
  "item_id": "337561634",
  "title": "40V HP Brushless 20 in. Cordless Battery Walk Behind Push Lawn Mower with 6.0 Ah Battery and Charger",
  "brand": "RYOBI",
  "model": "RY40HPLM06K",
  "price": 359,
  "was_price": 359,
  "priced_for_store": "910",
  "rating": 4.5112,
  "reviews": 1254,
  "url": "https://www.homedepot.com/p/RYOBI-40V-HP-Brushless-20-in-Cordless-Battery-Walk-Behind-Push-Lawn-Mower-with-6-0-Ah-Battery-and-Charger-RY40HPLM06K/337561634",
  "sponsored": false,
  "stock": [
   {"service": "boss", "store": "Secaucus", "store_id": "910", "qty": 531, "in_stock": true, "limited": false},
   {"service": "sth", "store": null, "store_id": "8119", "qty": 531, "in_stock": true, "limited": false},
   {"service": "express delivery", "store": "Secaucus", "store_id": "910", "qty": 423, "in_stock": true, "limited": false}
  ]
 },
 ...
]
```

With `QUERY = "cordless drill"` and `MAX_PAGES = 2` the search redirected client-side to a category page, and the block read GraphQL there. Page 2 is new items because `?Nao=` is added to the category path, not to `/s/<query>`:

```
landed on: https://www.homedepot.com/b/Tools-Power-Tools-Drills/Cordless/N-5yc1vZc27fZ1z140i3 | keyword: None
page 1: 24 products from graphql (0 sponsored, 0 repeats), 24 unique so far, store 6501, totalProducts=910
page 2: 30 products from graphql (10 sponsored, 0 repeats), 44 unique so far, store 6501, totalProducts=910
stopped: MAX_PAGES=2 cap hit; about 910 products in total, the rest NOT collected
rows whose title/brand/model contain a query word: 42/44
store 6501 Gulf Freeway | nearby: [('Pasadena', '565'), ('Pearland', '6543'), ('Eldorado-Webster (R0567)', '1859'), ('Gulfgate Mall', '6509')]
```

With the nonsense `QUERY = "qzxvbnmkqzx"` the site still answered with a grid (deck trim whose model numbers contain `ZX`); only the query-word count shows it is not a match:

```
landed on: https://www.homedepot.com/s/qzxvbnmkqzx | keyword: qzxvbnmkqzx
page 1: 24 products from ssr (0 sponsored, 0 repeats), 24 unique so far, store 6501, totalProducts=44
page 2: 0 products from ssr (0 sponsored, 0 repeats), 24 unique so far, store 6501, totalProducts=23
stopped: end of results (short page)
rows whose title/brand/model contain a query word: 0/24
```

### 2. Product page: specs + store availability (browser)

```python
import json, time

ITEM_ID = "320326875"   # the digits at the end of any /p/ URL, or item_id from the search rows

# Record the product page's own GraphQL call (request + response) before page scripts run.
HOOK = """
(() => {
  if (window.__gql) return;
  window.__gql = [];
  const orig = window.fetch;
  window.fetch = async function (input, init) {
    const url = String((input && input.url) || input);
    const rec = url.includes('/federation-gateway/graphql?opname=productClientOnlyProduct')
      ? {url, headers: (init && init.headers) || {}, body: init && init.body} : null;
    if (rec) window.__gql.push(rec);
    let res;
    try { res = await orig.apply(this, arguments); } catch (e) { if (rec) rec.err = String(e); throw e; }
    if (rec) { rec.status = res.status; try { rec.resp = await res.clone().text(); } catch (e) { rec.err = String(e); } }
    return res;
  };
})();
"""

READ = """(() => {
  const el = document.getElementById('thd-helmet__script--productStructureData');
  let ld = null; try { ld = el ? JSON.parse(el.textContent) : null; } catch (e) {}
  return {stale: !!window.__stale, ready: document.readyState, href: location.href, path: location.pathname,
          title: document.title, cookie: document.cookie.includes('THD_LOCALIZER='), ld,
          gql: (window.__gql || []).filter(r => r.status || r.err)};
})()"""

def peek(expr):
    """js() that returns None while a navigation swaps the document (the harness reports a CDP
    timeout from js() as RuntimeError('Runtime.evaluate timed out') and from cdp() as TimeoutError)."""
    try:
        return js(expr)
    except TimeoutError:
        return None
    except RuntimeError as e:
        if "timed out" in str(e) or "context" in str(e).lower():
            return None
        raise

def open_page(url, done, label, secs=45, tries=3):
    """Navigate, poll READ on the new document until done(data); retry the 403 'Error Page' or a
    timeout with backoff (20 s, 40 s), then raise."""
    for attempt in range(1, tries + 1):
        peek("window.__stale = true")
        try:
            goto_url(url)
        except TimeoutError:
            pass   # Page.navigate can outlast the harness's 5 s IPC timeout; the navigation still runs
        why = f"page not ready within {secs}s"
        for _ in range(secs):
            time.sleep(1)
            d = peek(READ)
            if not d or d["stale"] or d["ready"] != "complete":
                continue
            if "Error Page" in (d["title"] or ""):
                why = "403 'Error Page'"
                break
            if done(d):
                return d
        if attempt == tries:
            raise RuntimeError(f"{label}: {why} on {tries} tries ({url}). Home Depot rate-limits browser "
                               "sessions: stop and continue later in a new browser session.")
        print(f"{label}: {why}; retry {attempt}/{tries - 1} in {20 * attempt}s")
        time.sleep(20 * attempt)

def num(v):
    if isinstance(v, bool):
        return None
    if isinstance(v, (int, float)):
        return v
    if isinstance(v, str):
        try:
            return float(v)
        except ValueError:
            return None
    return None

def obj(v):
    return v if isinstance(v, dict) else {}

def arr(v):
    return v if isinstance(v, list) else []

def ld_product(ld):
    """The JSON-LD Product for ITEM_ID (dict, list or @graph), else None."""
    items = ld if isinstance(ld, list) else arr(obj(ld).get("@graph")) or [ld]
    for it in items:
        if isinstance(it, dict) and it.get("@type") == "Product" and str(it.get("productID")) == ITEM_ID:
            return it
    return None

def first_image(img):
    # schema.org image: a URL string, an ImageObject dict, or a list of either
    for x in (img if isinstance(img, list) else [img]):
        if isinstance(x, str) and x:
            return x
        if isinstance(x, dict) and isinstance(x.get("url") or x.get("contentUrl"), str):
            return x.get("url") or x.get("contentUrl")
    return None

def for_item(rec):
    try:
        return str(json.loads(rec.get("body") or "{}")["variables"]["itemId"]) == ITEM_ID
    except (ValueError, KeyError, TypeError):
        return False

tab = new_tab("about:blank")
cdp("Page.enable")
cdp("Page.addScriptToEvaluateOnNewDocument", source=HOOK)
# Warm-up: let the site pick a store (THD_LOCALIZER cookie) before the product page asks for stock.
open_page("https://www.homedepot.com/", lambda d: d["cookie"], "home page warm-up")
time.sleep(3)
# /p/<id> redirects to the canonical /p/<slug>/<id>; wait for that page's own JSON-LD.
d = open_page(f"https://www.homedepot.com/p/{ITEM_ID}",
              lambda d: d["path"].rstrip("/").endswith("/" + ITEM_ID) and ld_product(d["ld"]) is not None,
              "product page")
ld = ld_product(d["ld"])

# The page's productClientOnlyProduct response: specs + fulfillment for the store the site picked.
rec = None
for _ in range(30):
    rec = next((r for r in arr(peek("(window.__gql || []).filter(r => r.status || r.err)")) if for_item(r)), None)
    if rec:
        break
    time.sleep(1)
product, avail_error = {}, None
if not rec:
    avail_error = "no productClientOnlyProduct call for this item within 30s"
elif rec.get("status") != 200:
    # Seen: HTTP 206 {"data":{"Generic Errors API":null}} after about a dozen page loads in one session.
    avail_error = f"productClientOnlyProduct HTTP {rec.get('status')} {(rec.get('resp') or rec.get('err') or '')[:80]}"
else:
    try:
        product = obj(obj(json.loads(rec["resp"]).get("data")).get("product"))
    except (ValueError, TypeError):
        avail_error = "productClientOnlyProduct returned non-JSON"
    if not product and not avail_error:
        avail_error = "productClientOnlyProduct returned no product"
if avail_error:
    print("warning:", avail_error, "- availability is null; slow down or use a new session later")
    # Specs only: the server-rendered Apollo cache has the spec table but no store inventory.
    product = obj(peek(f"(window.__APOLLO_STATE__ || {{}})['base-catalog-{ITEM_ID}'] || null"))

def availability(p):
    out = []
    for opt in arr(obj(p.get("fulfillment")).get("fulfillmentOptions")):
        for svc in arr(obj(opt).get("services")):
            for loc in arr(obj(svc).get("locations")):
                inv = obj(obj(loc).get("inventory"))
                out.append({
                    "option": opt.get("type"),            # pickup / delivery
                    "service": svc.get("type"),           # bopis, boss, sth, express delivery
                    "store": loc.get("storeName"),
                    "store_id": loc.get("locationId"),
                    "qty": num(inv.get("quantity")),
                    "in_stock": inv.get("isInStock"),
                    "limited": inv.get("isLimitedQuantity"),
                    "out_of_stock": inv.get("isOutOfStock"),
                    "dates": svc.get("deliveryDates"),
                })
    return out

offers = ld.get("offers")
offers = obj(offers[0] if isinstance(offers, list) and offers else offers)
agg = obj(ld.get("aggregateRating"))
brand = ld.get("brand")
reviews = num(agg.get("reviewCount"))
groups = [g for g in arr(product.get("specificationGroup")) if isinstance(g, dict)]
detail = {
    "item_id": ITEM_ID,
    "title": ld.get("name"),
    "brand": brand.get("name") if isinstance(brand, dict) else brand if isinstance(brand, str) else None,
    "model": ld.get("model"),
    "store_sku": ld.get("sku"),
    "gtin13": ld.get("gtin13"),
    "price": num(offers.get("price")),
    "rating": num(agg.get("ratingValue")),
    "reviews": int(reviews) if reviews is not None else None,
    "url": offers.get("url"),
    "image": first_image(ld.get("image")),
    "specs": {s.get("specName"): s.get("specValue")
              for g in groups for s in arr(g.get("specifications")) if isinstance(s, dict)},
    "spec_groups": [g.get("specTitle") for g in groups],
    "availability": None if avail_error else availability(product),
    "availability_error": avail_error,
    "discontinued": obj(product.get("availabilityType")).get("discontinued"),
}
print(json.dumps(detail, indent=1)[:3500])
close_tab(tab)
```

Observed output (trimmed):

```
{
 "item_id": "320326875",
 "title": "Milwaukee M18 FUEL 18V Lithium-Ion Brushless Cordless 1/4 in. Hex Impact Driver (Tool-Only) 2953-20",
 "brand": "Milwaukee",
 "model": "2953-20",
 "store_sku": "1007901520",
 "gtin13": "0045242637867",
 "price": 179,
 "rating": 4.7,
 "reviews": 2491,
 "url": "https://www.homedepot.com/p/Milwaukee-M18-FUEL-18V-Lithium-Ion-Brushless-Cordless-1-4-in-Hex-Impact-Driver-Tool-Only-2953-20/320326875",
 "image": "https://images.thdstatic.com/productImages/3ca82630-d888-44f5-bd56-b2798a6d59f8/svn/milwaukee-impact-drivers-2953-20-64_100.jpg",
 "specs": {
  "Battery Voltage (V)": "18V",
  "Battery/Charger": "Tool Only",
  "Drive Size": "1/4 In. Hex",
  "Maximum Speed (RPM)": "3900 RPM",
  "Motor Type": "Brushless",
  "Torque (In-Lbs)": "2000",
  "Manufacturer Warranty": "5 Year Limited Warranty",
  "Product Width (in.)": "9.5 in",
  ... (22 specs)
 },
 "spec_groups": ["Details", "Warranty / Certifications", "Dimensions"],
 "availability": [
  {"option": "pickup", "service": "bopis", "store": "Secaucus", "store_id": "910",
   "qty": 0, "in_stock": false, "limited": false, "out_of_stock": true,
   "dates": {"startDate": "2026-10-23", ...}},
  {"option": "pickup", "service": "bopis", "store": "North Bergen", "store_id": "8475",
   "qty": 8, "in_stock": true, "limited": false, "out_of_stock": false,
   "dates": {"startDate": "2026-10-23", ...}},
  {"option": "delivery", "service": "express delivery", "store": null, "store_id": "8475",
   "qty": 8, "in_stock": true, "limited": false, "out_of_stock": false,
   "dates": {"startDate": "2026-10-23", ...}}
 ],
 "availability_error": null,
 "discontinued": false
}
```

### 3. Availability at other stores (browser, re-send the page's own request)

```python
import json, time
from email.utils import parsedate_to_datetime

ITEM_ID = "320326875"
STORE_IDS = ["121", "0628"]   # store ids from block 1 ("nearby") or block 2 (availability[].store_id)

HOOK = """
(() => {
  if (window.__gql) return;
  window.__gql = [];
  const orig = window.fetch;
  window.fetch = async function (input, init) {
    const url = String((input && input.url) || input);
    const rec = url.includes('/federation-gateway/graphql?opname=productClientOnlyProduct')
      ? {url, headers: (init && init.headers) || {}, body: init && init.body} : null;
    if (rec) window.__gql.push(rec);
    let res;
    try { res = await orig.apply(this, arguments); } catch (e) { if (rec) rec.err = String(e); throw e; }
    if (rec) { rec.status = res.status; try { rec.resp = await res.clone().text(); } catch (e) { rec.err = String(e); } }
    return res;
  };
})();
"""

def peek(expr):
    """js() that returns None on a harness timeout (js(): RuntimeError('Runtime.evaluate timed out'),
    cdp(): TimeoutError) or while a navigation swaps the document."""
    try:
        return js(expr)
    except TimeoutError:
        return None
    except RuntimeError as e:
        if "timed out" in str(e) or "context" in str(e).lower():
            return None
        raise

def open_page(url, check, label, secs=45, tries=3):
    """Navigate, poll the JS expression `check` on the new document; retry the 403 'Error Page' or a
    timeout with backoff (20 s, 40 s), then raise."""
    for attempt in range(1, tries + 1):
        peek("window.__stale = true")
        try:
            goto_url(url)
        except TimeoutError:
            pass   # Page.navigate can outlast the harness's 5 s IPC timeout; the navigation still runs
        why = f"page not ready within {secs}s"
        for _ in range(secs):
            time.sleep(1)
            d = peek("({stale: !!window.__stale, ready: document.readyState, title: document.title, ok: !!(" + check + ")})")
            if not d or d["stale"] or d["ready"] != "complete":
                continue
            if "Error Page" in (d["title"] or ""):
                why = "403 'Error Page'"
                break
            if d["ok"]:
                return
        if attempt == tries:
            raise RuntimeError(f"{label}: {why} on {tries} tries ({url}). Home Depot rate-limits browser "
                               "sessions: stop and continue later in a new browser session.")
        print(f"{label}: {why}; retry {attempt}/{tries - 1} in {20 * attempt}s")
        time.sleep(20 * attempt)

def retry_after_seconds(value, default):
    """Retry-After is either delay-seconds or an HTTP-date."""
    if not value:
        return default
    try:
        return max(0.0, float(value))
    except ValueError:
        pass
    try:
        return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())
    except (TypeError, ValueError):
        return default

def obj(v):
    return v if isinstance(v, dict) else {}

def arr(v):
    return v if isinstance(v, list) else []

def for_item(rec):
    try:
        return str(json.loads(rec.get("body") or "{}")["variables"]["itemId"]) == ITEM_ID
    except (ValueError, KeyError, TypeError):
        return False

def send(rec, body, label, tries=3):
    """POST the page's own request with a new body. Retries network errors, timeouts, 429 and 5xx
    with backoff (Retry-After honored when readable), raises on anything else (403/206 = blocked)."""
    for attempt in range(1, tries + 1):
        # Fire and poll, so a slow API call cannot hit the harness's 5 s js() timeout.
        peek("window.__send = null; fetch(" + json.dumps(rec["url"]) + ", {method: 'POST', headers: "
             + json.dumps(rec["headers"]) + ", body: " + json.dumps(json.dumps(body)) + ", credentials: 'include'})"
             ".then(async r => { window.__send = {status: r.status, retryAfter: r.headers.get('Retry-After'), text: await r.text()}; })"
             ".catch(e => { window.__send = {status: 0, err: String(e)}; }); 0")
        r = None
        for _ in range(30):
            time.sleep(1)
            r = peek("window.__send")
            if r:
                break
        status = (r or {}).get("status")
        if status == 200:
            try:
                p = (json.loads(r["text"]).get("data") or {}).get("product")
            except (ValueError, TypeError, AttributeError):
                p = None
            if isinstance(p, dict) and p:
                return p
            raise RuntimeError(f"{label}: HTTP 200 without a product: {r['text'][:160]}")
        if r and status not in (0, 429) and status < 500:
            raise RuntimeError(f"{label}: HTTP {status} {r.get('text', '')[:120]} (blocked or soft-blocked; "
                               "use a new browser session later)")
        if attempt == tries:
            raise RuntimeError(f"{label}: {('HTTP %s' % status) if r else 'no response in 30s'} on {tries} tries")
        wait = retry_after_seconds((r or {}).get("retryAfter"), 5 * 2 ** attempt)
        print(f"{label}: {('HTTP %s' % status) if r else 'no response'}; retry {attempt}/{tries - 1} in {wait:.0f}s")
        time.sleep(wait)

tab = new_tab("about:blank")
cdp("Page.enable")
cdp("Page.addScriptToEvaluateOnNewDocument", source=HOOK)
# Warm-up: let the site pick the session's store first.
open_page("https://www.homedepot.com/", "document.cookie.includes('THD_LOCALIZER=')", "home page warm-up")
time.sleep(3)
open_page(f"https://www.homedepot.com/p/{ITEM_ID}", f"location.pathname.endsWith('/{ITEM_ID}')", "product page")
rec = None
for _ in range(30):
    recs = peek("(window.__gql || []).filter(r => r.status || r.err)") or []
    rec = next((r for r in recs if isinstance(r, dict) and for_item(r)), None)
    if rec:
        break
    time.sleep(1)
if not rec:
    raise RuntimeError("no productClientOnlyProduct call for this item within 30s (layout change?)")
if rec.get("status") != 200:
    print("note: the page's own call answered HTTP", rec.get("status"), "- re-sent calls may fail the same way")

# Re-send the page's own request with a different storeId, the same thing the logged-out
# "change store" picker does. Only pickup rows follow the new storeId; delivery rows stay
# what the session's own store gets (session store printed here).
page_body = json.loads(rec["body"])
print("session store:", obj(page_body.get("variables")).get("storeId"))
for store_id in STORE_IDS:
    body = json.loads(rec["body"])
    body["variables"]["storeId"] = store_id
    p = send(rec, body, f"store {store_id}")
    for opt in arr(obj(p.get("fulfillment")).get("fulfillmentOptions")):
        for svc in arr(obj(opt).get("services")):
            for loc in arr(obj(svc).get("locations")):
                inv = obj(obj(loc).get("inventory"))
                print(store_id, opt.get("type"), svc.get("type"), loc.get("storeName"), loc.get("locationId"),
                      "qty", inv.get("quantity"), "in_stock", inv.get("isInStock"))
    print(store_id, "price", obj(p.get("pricing")).get("value"))
    time.sleep(3)   # polite pacing
close_tab(tab)
```

Observed output:

```
session store: 910
121 pickup bopis Cumberland 121 qty 6 in_stock True
121 delivery express delivery None 8475 qty 8 in_stock True
121 price 179.0
0628 pickup bopis San Carlos 0628 qty 2 in_stock True
0628 delivery express delivery None 8475 qty 8 in_stock True
0628 price 179.0
```

Only the pickup rows follow the requested `storeId`. The express-delivery row came back unchanged for every requested store: the same `locationId` that the session's own call returned in block 2. Store 121 (Cumberland) is the store fresh sessions were served before localizing (see Gotchas) and is far from the session store; the re-sent request still answered for it, so the store does not have to be in the nearby list.

## Fields

Search rows (block 1), one dict per non-sponsored product, deduped by `item_id` (rows without an id are kept):

| Key | Source | Example / notes |
|---|---|---|
| `item_id` | `itemId` | `"337561634"`, the number at the end of `/p/` URLs; `None` if missing |
| `title` | `identifiers.productLabel` | without brand and model |
| `brand` | `identifiers.brandName` | `"RYOBI"` |
| `model` | `identifiers.modelNumber` | `"RY40HPLM06K"` |
| `price` | `pricing.value` | number, store-specific; SSR key is `pricing({"isBrandPricingPolicyCompliant":false,"storeId":"..."})` |
| `was_price` | `pricing.original` | number |
| `priced_for_store` | `metadata.stores.storeId` of the same page | `"910"` |
| `rating` | `reviews.ratingsReviews.averageRating` | site sends a string; returned as float (`4.5112`) or `None` |
| `reviews` | `reviews.ratingsReviews.totalReviews` | site sends a string; returned as int or `None` |
| `url` | `identifiers.canonicalUrl` | made absolute |
| `sponsored` | `info.isSponsored` | always `False` in `rows` (sponsored items are skipped and counted) |
| `stock` | `fulfillment.fulfillmentOptions[].services[].locations[]` | list of `{service, store, store_id, qty, in_stock, limited}`; services seen: `bopis` (store pickup), `boss` (ship to store), `sth` (ship to home), `express delivery` |

Block 1 also prints the landing path, `searchReport.keyword`, per-page counts, the stop reason, the query-word match count, and `metadata.stores` (`storeId`, `storeName`, `nearByStores[]`).

Product detail (block 2):

| Key | Source | Example / notes |
|---|---|---|
| `item_id` | `ITEM_ID`, checked against JSON-LD `productID` | `"320326875"` |
| `title` | JSON-LD `name` | brand + label + model |
| `brand` | JSON-LD `brand.name` (or a plain string) | `"Milwaukee"` |
| `model` | JSON-LD `model` | `"2953-20"` |
| `store_sku` | JSON-LD `sku` | `"1007901520"` |
| `gtin13` | JSON-LD `gtin13` | `"0045242637867"` |
| `price` | JSON-LD `offers.price` | number |
| `rating` | JSON-LD `aggregateRating.ratingValue` | site sends `"4.7"`; returned as float |
| `reviews` | JSON-LD `aggregateRating.reviewCount` | int |
| `url` | JSON-LD `offers.url` | canonical `/p/<slug>/<id>` |
| `image` | JSON-LD `image` | first image URL (string or ImageObject handled) |
| `specs` | GraphQL `product.specificationGroup[].specifications[]` | `{specName: specValue}`, 22 for the example |
| `spec_groups` | `specificationGroup[].specTitle` | `["Details", "Warranty / Certifications", "Dimensions"]` |
| `availability` | GraphQL `product.fulfillment` | list of `{option, service, store, store_id, qty, in_stock, limited, out_of_stock, dates}`; `None` when the call failed |
| `availability_error` | | `None`, or why availability is missing (e.g. HTTP 206) |
| `discontinued` | `product.availabilityType.discontinued` | `false` |

Block 3 prints one line per stock location: requested store id, option, service, `storeName`, `locationId`, quantity, `isInStock`; then the price for that store.

## Gotchas

- **Install the fetch hook before navigating.** Blocks open `about:blank`, call `Page.addScriptToEvaluateOnNewDocument`, then navigate. A first version of the hook that pushed the record from an un-awaited `.then()` captured nothing in a background tab; awaiting `res.clone().text()` inside the wrapper (as in the blocks) worked in hidden tabs.
- **Read only the new document.** Before each navigation the blocks set `window.__stale` on the old page and skip any read where it is still set, then check the path (`/s/` or `/b/`, the same base path for later pages, `/<ITEM_ID>` for products) and, for products, that the JSON-LD `productID` and the GraphQL call's `variables.itemId` are the requested item.
- **Harness timeouts.** `goto_url()` and `js()` go through a 5 s IPC timeout. `cdp()` raises `TimeoutError`; `js()` re-raises it as `RuntimeError("Runtime.evaluate timed out ...")`. The blocks treat both as "not ready yet" inside their poll loops, and block 3 fires its re-sent `fetch` and polls for the result instead of awaiting it inside one `js()` call.
- **SSR vs GraphQL differs per page type.** Keyword pages (`/s/...`) had the full grid in `__APOLLO_STATE__`; category redirects had 12 server-rendered items and fetched the full 24 client-side. Block 1 waits for whichever is complete.
- **Sponsored and repeated items.** "lawn mower" pages held 24 to 28 products (2 to 5 sponsored) and pages 2 and 3 repeated 3 to 7 earlier item IDs (3 pages gave 59 to 63 unique rows, 60 in the run above); a "cordless drill" category page 2 held 30 with 10 sponsored. Block 1 skips `info.isSponsored` and dedupes by item ID. A short page (< 24 products) is treated as the end; a page of only repeats stops the run and says the rest was not collected.
- **`totalProducts` drifts.** For "lawn mower" it read between 1952 and 2033 across loads and once 6839 (on page 3 of one run); for "qzxvbnmkqzx" it read 31, 36, 44 and 23 across loads. Block 1 does not use it to decide the end.
- **No "no match" signal.** A nonsense query (`qzxvbnmkqzx`) returned HTTP 200, `/s/qzxvbnmkqzx`, `didYouMean: null`, `correctedKeyword: null` and a grid of unrelated deck trim (the page itself said "31 results"). Block 1 prints how many rows contain a query word in title, brand or model (0/24 there, most rows for real queries).
- **Prices and stock are per store.** The store is picked from the browser's exit IP and changed between sessions (stores in several states across sessions on the same day). Before the `THD_LOCALIZER` cookie exists, the server rendered store 121 (Cumberland) on the first page of fresh sessions; after the cookie was set, the SSR keys switched to the local store. So a run that skips the warm-up can mix stores across pages. Each response's `metadata.stores.storeId` matches its own rows: block 1 copies it into `priced_for_store`, and `stock[].store_id` / `availability[].store_id` carry the `locationId` of each stock line (`sth` lines use a distribution center id such as 8119, not a store).
- **`quantity: 0` with `isLimitedQuantity: true`** (and `isOutOfStock: false`) appeared for in-store pickup on some rows, so 0 is not always "out of stock". Check `isOutOfStock`.
- **Out of stock at the home store** can return a second pickup location. In the block 2 run above, Secaucus came back `qty 0`, `out_of_stock: true` plus North Bergen (8475) qty 8; an earlier session got its home store out of stock plus `San Carlos` qty 2 and the page tile read "Pickup Nearby ... In stock 19 mi away". Express delivery came from a nearby store (1952 for session store 1927, 8475 for 910) or from the session store itself (6501).
- **Page size is fixed.** Re-sending searchModel with `pageSize` 48 or 96 returned **HTTP 403 "Access Denied"** (later pageSize-24 calls in the same session still worked). `pageSize` 24 with `startIndex` 24/48/72/96 returned 200. Prefer page loads with `?Nao=`.
- **Block page detection.** A blocked session serves `<title>Error Page</title>` with HTTP 403 for every URL. The block is not always permanent: twice, the first search load of a fresh session got the Error Page and a later load in the same session worked (a few minutes later once, about a minute later today). So the blocks retry twice (20 s, then 40 s) and then raise; after that, stop and continue later in a new session.
- **Soft block on the product API.** After about a dozen page loads in one session, the product page's own `productClientOnlyProduct` call answered **HTTP 206** with `{"data":{"Generic Errors API":null},"error":[...]}` while the page itself still rendered. Block 2 then returns `availability: None` with `availability_error` set and takes the spec table from the Apollo cache (`base-catalog-<itemId>`, which has `specificationGroup` but no store inventory); block 3 raises on a 206. A new session returned 200 again.
- **Retry-After.** The blocks honor `Retry-After` (seconds or HTTP-date) when the browser can read it; on the cross-origin `apionline.homedepot.com` calls it may not be exposed, and the blocks fall back to fixed backoff.
- **First page of a fresh session.** `wait_for_load()` returned while the tab was still an empty document (no title, no body) for about 10 s before the real page arrived and `THD_LOCALIZER` was set. The blocks poll for the data they need (cookie, JSON-LD, GraphQL, `SearchModel`) instead of trusting `wait_for_load()`.
- **Hidden tab scrolling.** `scroll()` timed out (5 s) on a background tab; no scrolling is needed for the blocks above, and none of them brings the tab to the foreground.
- No cookie/consent banner appeared from a US exit IP. JSON-LD on search pages (`thd-helmet__script--browseSearchStructuredData`) is not a reliable grid source: in the raw HTML of the category redirect it listed 12 items per page, and after rendering it listed 24 on page 1 and 32 on page 2 while the grid showed 24.

## What does not work

- **Plain HTTP** (`curl`, `http_get`) to search and product pages: HTTP 403 "Error Page", with curl's own UA and with a desktop Chrome UA.
- **Bigger pages**: searchModel with `pageSize` > 24 returned 403 "Access Denied".
- **Heavy sessions**: after ~30 page loads in ~25 minutes, all pages in that session returned 403 until a new browser session was started.
- **Telling a real hit from the fallback grid**: the site sends no "no results" flag (see Gotchas).
- **Search JSON-LD as the only source**: its item count does not match the grid (see Gotchas), it has no model field (model is only appended to `name`), and 4 bundle items had no `brand`.
