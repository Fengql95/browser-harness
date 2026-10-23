# Temu — Category Product Listings

Field-tested against www.temu.com on 2026-10-23 using a Browser Use cloud browser (US session, prices in USD) and http_get.

## Quick summary

- **Logged-out keyword search is closed.** `search_result.html?search_key=...` redirected to `login.html` or to a `bgn_verification.html` captcha in all 8 attempts (8 separate browser sessions). The SEO search landers (`/<keyword>-<id>-s.html`) open, but showed "No results found. Suggested for you" with unrelated items.
- **Category pages (`/gc/...-<optId>.html`) are the working way to list products, but not in every session.** Some fresh browser sessions get a `bgn_verification.html` captcha on the very first `/gc/` navigation and on every `/gc/` page after it; other fresh sessions load them all. Every block below checks for that and stops. If it fires on the first page, close that browser and start a new session. Do not try to solve the captcha.
- When a session is let in, `/gc/` pages expose an ItemList JSON-LD block and a page store (`window.rawData.store.goodsList`). Each item has the goods ID, title, price, list price, sold count, rating, review count and URL.
- **Pagination works** with the page's own "See more items" button. Each click POSTs to `/api/seo/opt_rec_goods_list`, and the JSON response can be read through CDP. One run of 10 loads on `/gc/apparel-accessories/shoes-187.html` collected 274 unique products with no captcha.
- **Product detail pages need login.** Opening `...-g-<goods_id>.html` or `goods.html?goods_id=...` redirected to `login.html` or showed a captcha in all 8 attempts across 7 browser sessions. That included the first navigation of a brand-new browser. The "Quick look" button on a category card also returned `VERIFY_AUTH` and opened a captcha. The category data above is the per-product detail you can get logged out.
- **`http_get` alone does not work.** Category, search, product and home pages came back as about 250-320 KB app shells with no product data, both with `http_get`'s default `Mozilla/5.0` and with a truthful `Python-urllib/3.14` user agent (checked on a `/gc/` page). `sitemap-index.xml` returned 403 to plain HTTP.
- **Run the search and product checks (block 4) last.** In 2 of 2 sessions where search ran first, later `/gc/` pages came back as empty error pages (see Gotchas).

## What works

### 1. One category page: JSON-LD + page store

`open_page()` below is shared by every block. It marks the old document, navigates, and waits until a new document is `complete`, so it never reads the previous page's ready state. It returns the gate (`"login wall"` or `"captcha"`) when Temu redirected, `"error page"` when the requested URL loaded but the store has `isErrorPage: true`, and raises if it landed anywhere else.

```python
import json, time
from urllib.parse import unquote, urlparse

CATEGORY_URL = "https://www.temu.com/gc/electronics/communications/telephony/mobile-phone-accessories/mobile-phone-cases-2353.html"
GATES = {"/login.html": "login wall", "/bgn_verification.html": "captcha"}

def open_page(url, settle=5, timeout=30):
    """Navigate, wait for the NEW document, return (final_url, gate or None)."""
    js("window.__bhOld = 1")                    # disappears when the document is replaced
    goto_url(url)
    end = time.time() + timeout
    while True:
        try:
            if js("!window.__bhOld && document.readyState === 'complete'"):
                break
        except (RuntimeError, TimeoutError):    # evaluate raced the navigation; poll again
            pass
        if time.time() > end:
            raise TimeoutError(f"{url} did not load within {timeout}s")
        time.sleep(0.5)
    time.sleep(settle)                          # the product grid hydrates after readyState=complete
    final = page_info()["url"]
    path = unquote(urlparse(final).path)
    if path in GATES:
        return final, GATES[path]
    if path != unquote(urlparse(url).path):
        raise RuntimeError(f"asked for {url}, landed on {final}")
    if js("!!(window.rawData && window.rawData.store && window.rawData.store.isErrorPage)"):
        return final, "error page"              # soft block: requested URL, empty store
    return final, None

new_tab()
final, gate = open_page(CATEGORY_URL)
if gate:
    raise SystemExit(f"Gated ({gate}): {final[:120]}")

# 1) JSON-LD ItemList: name, url, price, availability, rating, review count
ld_products = []
for raw in js("[...document.querySelectorAll('script[type=\"application/ld+json\"]')].map(s => s.textContent)") or []:
    try:
        obj = json.loads(raw)
    except ValueError:
        continue
    nodes = obj if isinstance(obj, list) else obj.get("@graph", [obj]) if isinstance(obj, dict) else []
    for o in nodes:
        if isinstance(o, dict) and o.get("@type") == "ItemList":
            ld_products += [e for e in o.get("itemListElement") or []
                            if isinstance(e, dict) and e.get("@type") == "Product"]

# 2) Page store: same items plus goods ID, sold count, list price
store = js("""(() => {
  const s = window.rawData && window.rawData.store;
  if (!s || !Array.isArray(s.goodsList)) return null;
  return {total: s.total, hasMore: s.hasMore, pageSize: s.pageSize,
          goods: s.goodsList.map(g => g && g.data).filter(Boolean)};
})()""")
if not store or not store["goods"]:
    raise RuntimeError("no goodsList in window.rawData.store; the page layout may have changed")

print("title:", page_info()["title"])
print("JSON-LD products:", len(ld_products))
print("store goods:", len(store["goods"]), "| total:", store["total"], "| hasMore:", store["hasMore"])
for g in store["goods"][:3]:
    p = g.get("priceInfo") if isinstance(g.get("priceInfo"), dict) else {}
    c = g.get("comment") if isinstance(g.get("comment"), dict) else {}
    tip = g.get("salesTip") if isinstance(g.get("salesTip"), str) else ""
    print(g.get("goodsId"), "|", str(g.get("title") or "")[:60], "|", p.get("priceStr"),
          "|", tip.strip(), "|", c.get("goodsScore"), c.get("reviewCount"))
```

Observed output:

```
title: 🐴 Your Phone with Protection: Shop Our Durable Cases Now - Temu
JSON-LD products: 34
store goods: 34 | total: 1000 | hasMore: True
601099555963051 | 1/2/4/40pcs Safety Lock Summer New Pad Wrapped Around Thick  | 2.99 | + sold | 4.8 1978
601099531706019 | Transparent TPU Shockproof Case for iPhone 17 Air 16 15 14 1 | 1.74 | 200K+ sold | 4.7 1782
601099630644217 | Phone Case: for iPhone 17 18 Pro Max Case Upgrade Phone Case | 1.29 | + sold | 4.7 794
```

(The horse in the title is the harness tab marker, not Temu's.)

In a gated session the same block prints `Gated (captcha): https://www.temu.com/bgn_verification.html?verifyCode=...` and exits. That happened on the first navigation of 2 of 3 fresh browsers in one re-test; in a later pass, 3 of 3 fresh browsers were let in. Stop a gated browser and start a new one.

The JSON-LD `Product` entries hold `url`, `image`, `name`, `offers.price`, `offers.priceCurrency`, `offers.availability`, `aggregateRating.ratingValue`, `aggregateRating.reviewCount` and `brand.name`. The store has everything the JSON-LD has plus the goods ID, sold count and list price, so the scraper below uses the store.

### 2. Scrape a category with pagination ("See more items")

Run after block 1, in the same tab. Page 1 comes from the page store. Each "See more items" click fires `POST /api/seo/opt_rec_goods_list` (offset steps of 40), and the script reads that response through CDP. You cannot replay that endpoint yourself: the request carries per-session `Anti-Content` and `VerifyAuthToken` headers, so let the page make the call.

A load only counts when the click produced a finished `200` response with `"success": true`. A missing request, a network failure, a non-200 status (429/5xx: waits for `Retry-After`, in seconds or as an HTTP date, else backs off) or a timed-out CDP call is retried up to `TRIES` times per load and then raises. A gate mid-run raises too. `MAX_LOADS` is a safety cap: when it is hit while `has_more` is still true, the script says the list is partial. Only a successful response with `has_more: false` (or page 1 with `hasMore: false`) marks the category complete. No load failed in today's runs, so the retry path itself was not exercised against a real 429 or 5xx.

```python
import json, re, time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.parse import unquote, urljoin, urlparse

CATEGORY_URL = "https://www.temu.com/gc/electronics/communications/telephony/mobile-phone-accessories/mobile-phone-cases-2353.html"
MAX_LOADS = 6        # safety cap on "See more items" clicks; reported when hit
PAUSE_S = 4          # polite gap between loads
TRIES = 3            # attempts per load before raising
API = "/api/seo/opt_rec_goods_list"
GATES = {"/login.html": "login wall", "/bgn_verification.html": "captcha"}

def open_page(url, settle=5, timeout=30):
    """Navigate, wait for the NEW document, return (final_url, gate or None)."""
    js("window.__bhOld = 1")                    # disappears when the document is replaced
    goto_url(url)
    end = time.time() + timeout
    while True:
        try:
            if js("!window.__bhOld && document.readyState === 'complete'"):
                break
        except (RuntimeError, TimeoutError):    # evaluate raced the navigation; poll again
            pass
        if time.time() > end:
            raise TimeoutError(f"{url} did not load within {timeout}s")
        time.sleep(0.5)
    time.sleep(settle)
    final = page_info()["url"]
    path = unquote(urlparse(final).path)
    if path in GATES:
        return final, GATES[path]
    if path != unquote(urlparse(url).path):
        raise RuntimeError(f"asked for {url}, landed on {final}")
    if js("!!(window.rawData && window.rawData.store && window.rawData.store.isErrorPage)"):
        return final, "error page"              # soft block: requested URL, empty store
    return final, None

def gated():
    gate = GATES.get(unquote(urlparse(page_info()["url"]).path))
    if gate:
        return gate
    dlg = js("(() => { const d = document.querySelector('[role=dialog]'); return d ? d.innerText : '' })()") or ""
    return "captcha dialog" if "Security Verification" in dlg else None

def first(o, *keys):
    return next((o[k] for k in keys if o.get(k) not in (None, "")), None)

def as_dict(v):
    return v if isinstance(v, dict) else {}

def as_str(v):
    return v.strip() or None if isinstance(v, str) else None

def as_num(v):
    return v if isinstance(v, (int, float)) and not isinstance(v, bool) else None

def image_url(v):                               # str, {"url": ...} or a list of either
    if isinstance(v, list):
        v = v[0] if v else None
    if isinstance(v, dict):
        v = v.get("url") or v.get("contentUrl")
    return as_str(v)

def norm(g):
    """Page-1 store items are camelCase, the See-more API is snake_case."""
    price = as_dict(first(g, "priceInfo", "price_info"))
    comment = as_dict(g.get("comment"))
    gid = first(g, "goodsId", "goods_id")
    sold = as_str(first(g, "salesTip", "sales_tip"))
    link = as_str(first(g, "seoLinkUrl", "seo_link_url"))
    lp = as_str(first(price, "marketPriceStr", "market_price_str"))
    return {
        "goods_id": str(gid) if isinstance(gid, (int, str)) else None,
        "title": as_str(g.get("title")),
        "price": as_str(first(price, "priceStr", "price_str")),
        "list_price": lp if lp and re.search(r"[1-9]", lp) else None,   # "0.00" means no list price shown
        "currency": as_str(price.get("currency")),
        "sold": sold if sold and re.search(r"\d", sold) else None,   # " + sold" means no count shown
        "rating": as_num(first(comment, "goodsScore", "goods_score")),
        "reviews": as_num(first(comment, "reviewCount", "review_count")),
        "image": image_url(g.get("image")) or as_str(first(g, "thumbUrl", "thumb_url")),
        "url": urljoin("https://www.temu.com/", link) if link else None,
    }

items, seen = [], set()
def add(raw_list):
    n = 0
    for g in raw_list if isinstance(raw_list, list) else []:
        if not isinstance(g, dict):
            continue
        it = norm(g)
        if it["goods_id"] is not None:         # rows without an id are kept, not deduped
            if it["goods_id"] in seen:
                continue
            seen.add(it["goods_id"])
        items.append(it)
        n += 1
    return n

def click_see_more():
    for n in cdp("Accessibility.getFullAXTree")["nodes"]:
        if (n.get("role") or {}).get("value") == "button" and \
           str((n.get("name") or {}).get("value", "")).startswith("See more items"):
            b = n["backendDOMNodeId"]
            cdp("DOM.scrollIntoViewIfNeeded", backendNodeId=b)
            time.sleep(1)
            q = cdp("DOM.getBoxModel", backendNodeId=b)["model"]["content"]
            click_at_xy(sum(q[0::2]) / 4, sum(q[1::2]) / 4)
            return True
    return False

def read_load(timeout=20):
    """Wait for the API call the click started: (status, headers, body or None), or None if none was sent."""
    reqs, end = {}, time.time() + timeout
    while time.time() < end:
        for e in drain_events():
            m, p = e.get("method"), e.get("params") or {}
            rid = p.get("requestId")
            if m == "Network.requestWillBeSent" and API in p.get("request", {}).get("url", ""):
                reqs[rid] = (0, {})
            elif rid in reqs and m == "Network.responseReceived":
                reqs[rid] = (p["response"].get("status", 0), p["response"].get("headers") or {})
            elif rid in reqs and m == "Network.loadingFailed":
                return 0, {}, None
            elif rid in reqs and m == "Network.loadingFinished":
                try:
                    body = json.loads(cdp("Network.getResponseBody", requestId=rid)["body"])
                except (ValueError, KeyError, RuntimeError, TimeoutError):
                    body = None
                return reqs[rid][0], reqs[rid][1], body
        time.sleep(0.3)
    return (0, {}, None) if reqs else None

def retry_after(headers, attempt):
    v = next((str(x).strip() for k, x in headers.items() if k.lower() == "retry-after"), "")
    if v.isdigit():
        return min(float(v), 120)
    try:
        return min(max(0.0, (parsedate_to_datetime(v) - datetime.now(timezone.utc)).total_seconds()), 120)
    except (TypeError, ValueError, IndexError):
        return 5.0 * 2 ** attempt

final, gate = open_page(CATEGORY_URL)
if gate:
    raise SystemExit(f"Gated before page 1 ({gate}): {final[:160]}")

page1 = js("""(() => { const s = window.rawData && window.rawData.store;
  if (!s || !Array.isArray(s.goodsList)) return null;
  return {hasMore: s.hasMore, goods: s.goodsList.map(g => g && g.data).filter(Boolean)}; })()""")
if not page1 or not page1["goods"]:
    raise RuntimeError("no goodsList in window.rawData.store; the page layout may have changed")
print("page 1:", add(page1["goods"]), "items")
has_more = page1["hasMore"] is True

cdp("Network.enable")
loads = 0
while has_more and loads < MAX_LOADS:
    res = None
    for attempt in range(TRIES):
        drain_events()
        got, why = None, None
        try:
            if click_see_more():
                got = read_load()
            else:
                why = "no 'See more items' button"
        except TimeoutError as ex:              # a CDP call timed out; retry the click
            why = f"timeout: {ex}"
        gate = gated()
        if gate:
            raise RuntimeError(f"gated ({gate}) after {len(items)} items; the list is partial")
        if got is None:
            why = why or "the click sent no API request"
        elif got[0] == 200 and isinstance(got[2], dict) and got[2].get("success") is True:
            res = as_dict(got[2].get("result"))
            break
        else:
            why = f"HTTP {got[0]}, body {str(got[2])[:80]}"
        wait = retry_after(got[1] if got else {}, attempt)
        print(f"load {loads + 1} try {attempt + 1} failed ({why}); retrying in {wait:.0f}s")
        time.sleep(wait)
    else:
        raise RuntimeError(f"load {loads + 1} failed {TRIES} times after {len(items)} items; the list is partial")
    loads += 1
    batch = as_dict(res.get("data")).get("goods_list")
    batch = batch if isinstance(batch, list) else []
    new = add(batch)
    has_more = res.get("has_more") is True
    print(f"load {loads}: {len(batch)} returned, {new} new (total {len(items)}), has_more={has_more}")
    if has_more:
        time.sleep(PAUSE_S)

if has_more:
    print(f"STOPPED at the MAX_LOADS={MAX_LOADS} cap with has_more=True: these {len(items)} items are a partial list")
else:
    print("category complete: the last response said has_more=False")
print(len(items), "items,", sum(1 for x in items if x["goods_id"] is None), "without a goods ID")
print(json.dumps(items[:2], indent=1))
print("with sold count:", sum(1 for x in items if x["sold"]), "| with rating:", sum(1 for x in items if x["rating"] is not None))
```

Observed output from one run (trimmed). Per-load counts vary between runs: another run of the same code got 26 returned / 13 new on load 1 and 122 items in total.

```
page 1: 34 items
load 1: 33 returned, 20 new (total 54), has_more=True
load 2: 21 returned, 13 new (total 67), has_more=True
load 3: 23 returned, 17 new (total 84), has_more=True
load 4: 26 returned, 19 new (total 103), has_more=True
load 5: 22 returned, 12 new (total 115), has_more=True
load 6: 19 returned, 14 new (total 129), has_more=True
STOPPED at the MAX_LOADS=6 cap with has_more=True: these 129 items are a partial list
129 items, 0 without a goods ID
[
 {
  "goods_id": "601099555963051",
  "title": "1/2/4/40pcs Safety Lock Summer New Pad Wrapped Around Thick Bag, Can Protect *-inch Phones, Suitable for Swimming, ...",
  "price": "2.99",
  "list_price": "28.46",
  "currency": "USD",
  "sold": null,
  "rating": 4.8,
  "reviews": 1978,
  "image": "https://img.kwcdn.com/product/fancy/90d0a5ba-a93b-4956-a955-4017404b9257.jpg",
  "url": "https://www.temu.com/1-2-4pcs-new-waterproof-cushion-surround-thickened-waterproof-bag-protects--inch-phones-...-g-601099555963051.html"
 },
 {
  "goods_id": "601099531706019",
  "title": "Transparent TPU Shockproof Case for iPhone 17 Air 16 15 14 13 12 11 Pro * Mini With Lens Protector Plating Candy Color Cover",
  "price": "1.74",
  "list_price": "12.99",
  "currency": "USD",
  "sold": "200K+ sold",
  "rating": 4.7,
  "reviews": 1782,
  "image": "https://img.kwcdn.com/product/Fancyalgo/VirtualModelMatting/c05aaca40d0135fe864f01c0badfb9f9.jpg",
  "url": "https://www.temu.com/transparent-clear-tpu-shockproof-case-for-iphone-16-15-14-13-12-11-pro--mini-with-lens-protector-plating-candy-color-cover-g-601099531706019.html"
 }
]
with sold count: 113 | with rating: 129
```

In a session that Temu gates, the script stops at the first check instead of returning an empty list:

```
Gated before page 1 (captcha): https://www.temu.com/bgn_verification.html?verifyCode=...&from=https%3A%2F%2Fwww.temu.com%2Fgc%2Felectronics%2F...
```

At larger scale, the same pagination loop (an earlier version of this block, without the retry and cap reporting) with `MAX_LOADS = 10` on `https://www.temu.com/gc/apparel-accessories/shoes-187.html` returned 35 items on page 1 plus 19-27 new items per load, 274 unique products in total. `has_more` was still `True`, and no captcha appeared.

### 3. Find category URLs (the substitute for keyword search)

Every `/gc/` page carries the top two levels of the catalog in `categoryData.categoryList`. Its own direct subcategories are in `categoryPageData.optList`, so you drill down one level per page.

```python
import time
from urllib.parse import unquote, urljoin, urlparse

START = "https://www.temu.com/gc/electronics-222.html"
GATES = {"/login.html": "login wall", "/bgn_verification.html": "captcha"}

def open_page(url, settle=5, timeout=30):
    """Navigate, wait for the NEW document, return (final_url, gate or None)."""
    js("window.__bhOld = 1")                    # disappears when the document is replaced
    goto_url(url)
    end = time.time() + timeout
    while True:
        try:
            if js("!window.__bhOld && document.readyState === 'complete'"):
                break
        except (RuntimeError, TimeoutError):    # evaluate raced the navigation; poll again
            pass
        if time.time() > end:
            raise TimeoutError(f"{url} did not load within {timeout}s")
        time.sleep(0.5)
    time.sleep(settle)
    final = page_info()["url"]
    path = unquote(urlparse(final).path)
    if path in GATES:
        return final, GATES[path]
    if path != unquote(urlparse(url).path):
        raise RuntimeError(f"asked for {url}, landed on {final}")
    if js("!!(window.rawData && window.rawData.store && window.rawData.store.isErrorPage)"):
        return final, "error page"              # soft block: requested URL, empty store
    return final, None

final, gate = open_page(START)
if gate:
    raise SystemExit(f"Gated ({gate}): {final[:120]}")

store = js("""(() => { const s = window.rawData && window.rawData.store; if (!s) return null;
  return {tree: s.categoryData && s.categoryData.categoryList,
          children: s.categoryPageData && s.categoryPageData.optList}; })()""") or {}
if not isinstance(store.get("tree"), list) or not store["tree"]:
    raise RuntimeError("no categoryData.categoryList in the page store; the page layout may have changed")

def link(n):
    u = n.get("seoLinkUrl")
    return urljoin("https://www.temu.com/", u) if isinstance(u, str) and u else None

# Top two levels of the whole catalog (same on every /gc/ page)
cats = []
def walk(nodes, path):
    for n in nodes if isinstance(nodes, list) else []:
        if not isinstance(n, dict):
            continue
        name = str(n.get("optName") or "").strip()
        if link(n):
            cats.append({"opt_id": n.get("optId"), "path": " > ".join(path + [name]), "url": link(n)})
        walk(n.get("childOpts"), path + [name] if link(n) else path)
walk(store["tree"], [])
print(len(cats), "top-level + level-2 categories")
for c in cats[:3]:
    print(c)

# Direct subcategories of the current page: drill down level by level
children = store.get("children") if isinstance(store.get("children"), list) else []
subs = [{"opt_id": o.get("optId"), "name": str(o.get("optName") or "").strip(), "url": link(o)}
        for o in children if isinstance(o, dict) and link(o)]
print(len(subs), "subcategories of", START)
for s in subs[:5]:
    print(s)
```

Observed output (trimmed):

```
178 top-level + level-2 categories
{'opt_id': 1, 'path': 'Animals & Pet Supplies', 'url': 'https://www.temu.com/gc/animals-pet-supplies-1.html'}
19 subcategories of https://www.temu.com/gc/electronics-222.html
{'opt_id': 340, 'name': 'Marine Electronics', 'url': 'https://www.temu.com/gc/electronics/marine-electronics-340.html'}
{'opt_id': 342, 'name': 'Networking', 'url': 'https://www.temu.com/gc/electronics/networking-342.html'}
```

In a gated session it exits with `Gated (captcha): https://www.temu.com/bgn_verification.html?...` or `Gated (error page): ...` instead of printing 0 categories.

### 4. Check what is gated (search and product pages)

Run this last in a session; the search redirect can close `/gc/` afterwards.

```python
import time
from urllib.parse import quote, unquote, urljoin, urlparse

GATES = {"/login.html": "login wall", "/bgn_verification.html": "captcha"}

def open_page(url, settle=5, timeout=30):
    """Navigate, wait for the NEW document, return (final_url, gate or None)."""
    js("window.__bhOld = 1")                    # disappears when the document is replaced
    goto_url(url)
    end = time.time() + timeout
    while True:
        try:
            if js("!window.__bhOld && document.readyState === 'complete'"):
                break
        except (RuntimeError, TimeoutError):    # evaluate raced the navigation; poll again
            pass
        if time.time() > end:
            raise TimeoutError(f"{url} did not load within {timeout}s")
        time.sleep(0.5)
    time.sleep(settle)
    final = page_info()["url"]
    path = unquote(urlparse(final).path)
    if path in GATES:
        return final, GATES[path]
    if path != unquote(urlparse(url).path):
        raise RuntimeError(f"asked for {url}, landed on {final}")
    if js("!!(window.rawData && window.rawData.store && window.rawData.store.isErrorPage)"):
        return final, "error page"              # soft block: requested URL, empty store
    return final, None

def state(gate):
    if gate:
        return gate
    dlg = js("(() => { const d = document.querySelector('[role=dialog]'); return d ? d.innerText : '' })()") or ""
    return "captcha" if "Security Verification" in dlg else "open"

# Product page (logged out): take the first product from a category page.
# Do this before the search check; a search redirect can turn later /gc/ pages into error pages.
final, gate = open_page("https://www.temu.com/gc/electronics/communications/telephony/mobile-phone-accessories/mobile-phone-cases-2353.html")
cat_state = state(gate)
link = None if gate else js("""(() => { const s = window.rawData && window.rawData.store;
  const g = s && Array.isArray(s.goodsList) && s.goodsList[0];
  return g && g.data && typeof g.data.seoLinkUrl === 'string' ? g.data.seoLinkUrl : null; })()""")
if cat_state != "open" or not link:
    print("category page ->", cat_state, "| no product link, skipping the product check")
else:
    final, gate = open_page(urljoin("https://www.temu.com/", link))
    print("product page ->", state(gate), "|", final[:70])

# Keyword search (logged out)
final, gate = open_page("https://www.temu.com/search_result.html?search_key=" + quote("phone case", safe=""))
print("search_result.html ->", state(gate), "|", final[:70])
```

Observed output (fresh session where `/gc/` was let in, after blocks 1-3):

```
product page -> captcha | https://www.temu.com/1-2-4pcs-new-waterproof-cushion-surround-thickene
search_result.html -> login wall | https://www.temu.com/login.html?from=https%3A%2F%2Fwww.temu.com%2Fsear
```

The product URL stays the same and a "Security Verification" dialog covers it. In other sessions `search_result.html` went to `bgn_verification.html` (captcha) instead of `login.html`. When `/gc/` itself is gated, the block prints `category page -> captcha | no product link, skipping the product check` (or `-> error page`) and still runs the search check. With the search check first (an earlier version of this block), the category page then came back as `error page` in 2 of 2 sessions.

Stop at that point. Solving the captcha or signing in is out of scope for logged-out scraping.

## Fields

What `norm()` in block 2 returns, one dict per product. Examples are the second item of the block 2 output above.

| Key | Type | Page-1 store (`rawData.store.goodsList[].data`) | See-more API (`result.data.goods_list[]`) | Example |
|---|---|---|---|---|
| `goods_id` | str or None | `goodsId` (int) | `goods_id` (int) | `"601099531706019"` |
| `title` | str or None | `title` | `title` | `"Transparent TPU Shockproof Case for iPhone 17 Air ..."` |
| `price` | str or None | `priceInfo.priceStr` | `price_info.price_str` | `"1.74"` |
| `list_price` | str or None | `priceInfo.marketPriceStr` | `price_info.market_price_str` | `"12.99"`; `null` when the raw value is `"0.00"` (no list price shown) |
| `currency` | str or None | `priceInfo.currency` | `price_info.currency` | `"USD"` |
| `sold` | str or None | `salesTip` | `sales_tip` | `"200K+ sold"`; `null` when the raw value is `" + sold"` |
| `rating` | number or None | `comment.goodsScore` | `comment.goods_score` | `4.7` |
| `reviews` | number or None | `comment.reviewCount` | `comment.review_count` | `1782` |
| `image` | str or None | `image.url`, else `thumbUrl` | `image.url`, else `thumb_url` | `"https://img.kwcdn.com/product/Fancyalgo/...jpg"` |
| `url` | str or None | `seoLinkUrl` (relative) | `seo_link_url` (relative) | `"https://www.temu.com/transparent-clear-tpu-...-g-601099531706019.html"` |

Raw fields that `norm()` does not return but the items carry: price in cents (`priceInfo.price` / `price_info.price`, e.g. `174`), list price in cents (`marketPrice`, `0` when none), video (`video.videoUrl` / `video.video_url`), availability (`offerInfo.availability` / `offer_info.availability`), brand (`brandInfo.name` / `brand_info.name`).

The goods ID is also the number after `-g-` in every product URL. Category-level fields on the store: `total` (1000 on every category page checked), `pageSize` (40), `hasMore`, `pagination`, `query.opt_id`, and `isErrorPage` (true on a soft-blocked page).

## Gotchas

- **The browser is required.** `http_get` on `/gc/` pages returns an app shell with no JSON-LD or goods data. The product data only appears after the page's JavaScript runs.
- **Wait for hydration.** Read the store about 5 s after `wait_for_load()`. Page 1 held 34-40 items across the categories tested, not a fixed 40.
- **"See more items" is a button, not a URL.** The page URL does not change and there is no `?page=` parameter. Load more by clicking the accessibility-tree button named "See more items" and reading the API response. `window.rawData.store.goodsList` does not grow after a click. The new cards only appear in the DOM and in the network response.
- **Each load returns up to 40 items, often fewer, and overlaps earlier ones.** The request asks for `pageSize: 40`. On Mobile Phone Cases each load returned 19-33 items, of which only 12-20 were new, and the numbers differed between runs of the same code. On Shoes one load returned exactly 40 (25 new) and 19-27 per load were new. Dedupe by goods ID.
- **`list_price` is often absent, and it moves.** `marketPriceStr` is `"0.00"` (with `marketPrice: 0`) when no list price is shown, on 18 of 34 page-1 items on Mobile Phone Cases in one session; `norm()` maps that to `None`. The same item showed a list price of `28.46` in two sessions and `5.45` in another on the same day, and the second item's `12.99` was `"0.00"` in that session, so do not treat it as a stable reference price.
- **`sold` is often hidden.** About 12-18% of items had `salesTip == " + sold"` (16 of 129 on Mobile Phone Cases, 49 of 274 on Shoes) (no number), so the scraper maps that to `None`. `total` reports 1000 for each category tested, which looks like a cap rather than a real count.
- **Some words are replaced with `*`.** Titles, JSON-LD names and category names have words swapped for `*` or blanked, for example `"Pro * Mini"`, `"Can Protect *-inch Phones"`, and a category named `" "` (`/gc/electronics/communications-262.html`). Some slugs are trimmed the same way (`/gc/electronics/-scan-fax-345.html`). Treat titles as approximate, and do not rebuild URLs from names; use `seoLinkUrl`.
- **Category tree depth.** `categoryData.categoryList` has only 178 nodes (two levels). Deeper leaf categories such as Mobile Phone Cases (2353) only appear in their parent's `categoryPageData.optList`.
- **Region and currency.** The cloud session was served USD prices with `tracking.biz_region` 211. Temu localizes by visitor IP, so prices, currency and assortment may differ in another region.
- **`/gc/` access is decided per browser session, up front.** Some fresh sessions are sent to `bgn_verification.html` ("Security verification") on their very first `/gc/` page, before any other request, and every later `/gc/` page in that session is gated too (checked on 2 category URLs). Other fresh sessions load every `/gc/` page. In one re-test 2 of 3 fresh browsers were gated on the first page, an earlier reviewer pass saw 1 of 2, and a later pass saw 0 of 3. It did not depend on pacing. The fix is to stop that browser and start a new session, not to retry in the same one. The captcha page still defines `window.rawData.store` (without `goodsList`), so always check the URL before reading the store.
- **Pacing.** In sessions that were let in, 10 loads at about 5 s each (4 s pause plus 1 s scroll) on one category, plus more than a dozen `/gc/` page loads, triggered no new block. Keep a few seconds between actions, and stop if `gated()` returns anything.
- **Hitting search can close `/gc/` for the rest of the session.** In one session, after the search and product pages showed a captcha, `/gc/electronics-222.html` still returned the full tree. In 2 others, after `search_result.html` redirected to `login.html`, the next `/gc/` page loaded at the requested URL but with `rawData.store.isErrorPage: true`. In the session inspected closely, 2 such pages had an empty `goodsList`, an empty category tree and a generic "Temu | Shop for ..." title, and a third `/gc/` page then went to `bgn_verification.html`. `open_page()` reports that state as `"error page"`. Scrape categories first and run block 4 last, or in a separate browser.
- **robots.txt.** The `User-agent: *` group has no `Disallow` lines. Named AI and search crawlers get a list of disallowed account, checkout and filter-parameter paths. `/gc/` and product URLs are not disallowed. Do not send a crawler user agent.

## What does not work

| Attempt | Observed on 2026-10-23 |
|---|---|
| `/gc/` category pages in some fresh sessions | `bgn_verification.html` captcha on the first navigation and on every later `/gc/` page in that session (2 of 3 fresh browsers in one re-test, 0 of 3 in a later one). A new browser session got through |
| `/gc/` pages after `search_result.html` redirected to `login.html` (2 sessions) | Requested URL, but `isErrorPage: true` and no goods or categories; in one session a third `/gc/` page was a captcha |
| `search_result.html?search_key=phone%20case` in the browser | Redirect to `login.html?...login_scene=2...` (4 sessions), `bgn_verification.html` captcha (4 sessions) |
| SEO search lander `/phone-case-for-15-5040180202396-s.html` | Redirect to `bmw.html?...&search_key=phone+case+for+15`. The store title was "No results found. Suggested for you" with unrelated items, and the same happened for `search_key=wireless+earbuds` |
| Product page `...-g-<goods_id>.html` | Redirect to `login.html` (2 times, including the first navigation of a fresh browser) or a captcha (5 times) |
| `goods.html?goods_id=<id>` | Redirect to `login.html` in a fresh browser |
| "Quick look" on a category card | `POST /api/oak/integration/sku` returned `{"error_code":54001,"error_msg":"VERIFY_AUTH"}` and a captcha dialog |
| Homepage `https://www.temu.com/` in the browser | Slider or image captcha (`bgn_verification.html`) in 2 of 2 sessions where it was opened after other pages |
| `http_get` on home, search, `/gc/`, `-s.html`, product pages | 260-320 KB app shells with no product data or JSON-LD |
| `https://www.temu.com/sitemap-index.xml` via plain HTTP | 403 |
| Calling `/api/seo/opt_rec_goods_list` directly | Not attempted: it needs per-session `Anti-Content` and `VerifyAuthToken` headers generated by the page |

So logged-out you cannot get keyword search results, full product descriptions, SKU or variant pricing, shipping estimates, seller info or review text. What you can get is the category listing data above for any product that appears in a category.
