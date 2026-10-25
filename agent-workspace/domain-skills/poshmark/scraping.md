# Poshmark — Browse listings and extract title, brand, size, price, original price, URL

Field-tested against poshmark.com on 2026-10-24 using a Browser Use cloud browser and plain Python `urllib`.

robots.txt (fetched 2026-10-24) disallows `/search`, `/api`, `/listings` and `/users` for `User-agent: *`; everything below only requests `/brand/...` and `/category/...` browse pages, which it allows.

## Quick summary

- Brand and category browse pages (`/brand/Levi's-Women-Jeans`, `/category/Men-Shoes`) are server-rendered. Every page embeds `window.__INITIAL_STATE__` with 48 full listing objects, plus a JSON-LD `ItemList` with the 48 canonical listing URLs.
- Plain HTTP with an honest User-Agent returned HTTP 200 for every browse page tried (about 50 requests, 2 s apart, no 429, no captcha). No browser needed for the main task.
- Pagination is `?max_id=N` (N = page number). This is the URL the site's own "Next" button produces. With `sort_by=added_desc`, consecutive pages shared 0 or 1 IDs (new listings push rows onto the next page); the code de-duplicates by `id`.
- Filters are plain query params: `size[]=28`, `price[]=25-50`, `brand[]=Nike` (on category pages), `sort_by=...`, `availability=available`.
- Results are capped at 105 pages (about 5000 rows): `more.total` reports 5000 for any large result set. On `/brand/Levi's-Women-Jeans?sort_by=added_desc`, page 104 had `is_next_max_id_present: true`, page 105 had 48 rows and `false`, and pages 106, 107, 120 and 200 returned 0 rows. With the default sort, pages 106, 120 and 200 still returned 48 rows each (no overlap with page 105) but with `false`, and default-sort rows reshuffle between requests anyway. Treat the cap as a hard stop: stop when `has_next` is false, and when `total` is 5000 the code returns status `truncated_site_cap` instead of `complete`. Hitting your own `max_pages` while `has_next` is true returns `truncated_max_pages`.
- Keyword search lives under `/search`, which robots.txt disallows, so it was not requested. Use brand/category pages plus filters, then filter titles locally if you need keywords.

## What works

### 1. Plain HTTP, multi-page, with filters (main method)

```python
import gzip, json, re, time, urllib.error, urllib.parse, urllib.request

UA = "poshmark-listing-export/1.0 (python-urllib; personal research script)"
BASE = "https://poshmark.com"
RETRYABLE = {429, 500, 502, 503, 504}
SITE_CAP_TOTAL = 5000   # more.total never exceeds this; has_next turns false at page 105
EMAIL = re.compile(r"[\w.+-]+@[\w-]+\.[\w.-]+")
PHONE = re.compile(r"(?<!\d)(?:\+?1[\s.-]?)?\(?\d{3}\)?[\s.-]?\d{3}[\s.-]?\d{4}(?!\d)")

def redact(text):
    """Titles are seller-written free text: mask any email address or phone number in them."""
    return PHONE.sub("[phone]", EMAIL.sub("[email]", text))

def fetch(url, budget_s=120):
    """GET with capped exponential backoff under ONE deadline. Raises on 403/404 or when the budget is spent."""
    deadline, delay, attempt = time.monotonic() + min(budget_s, 120), 5, 0
    while True:
        attempt += 1
        remaining = deadline - time.monotonic()
        try:
            req = urllib.request.Request(url, headers={
                "User-Agent": UA, "Accept-Encoding": "gzip", "Accept-Language": "en-US,en;q=0.9"})
            with urllib.request.urlopen(req, timeout=max(0.1, min(30, remaining))) as r:
                body = r.read()
                if r.headers.get("Content-Encoding") == "gzip":
                    body = gzip.decompress(body)
                return r.geturl(), body.decode("utf-8")
        except urllib.error.HTTPError as e:
            if e.code not in RETRYABLE:
                raise
            err = e
        except (urllib.error.URLError, TimeoutError) as e:
            err = e
        if time.monotonic() + delay >= deadline:   # no sleep after the last attempt
            raise RuntimeError(f"giving up on {url} after {attempt} attempts: {err}")
        time.sleep(delay)
        delay *= 2

def browse_url(path, filters=None, page=1):
    """path: 'brand/<Brand>-<Dept>-<Category>' or 'category/<Dept>-<Category>' (both allowed by robots.txt)."""
    q = list((filters or {}).items())
    if page > 1:
        q.append(("max_id", page))
    url = f"{BASE}/{urllib.parse.quote(path, safe='/-_')}"
    return url + ("?" + urllib.parse.urlencode(q, doseq=True) if q else "")

def parse_page(html):
    marker = "window.__INITIAL_STATE__="
    i = html.find(marker)
    if i < 0:
        raise RuntimeError("no __INITIAL_STATE__ in page (layout changed or not a listings page)")
    state = json.JSONDecoder().raw_decode(html, i + len(marker))[0]
    if state.get("showCaptcha"):
        raise RuntimeError("Poshmark served a captcha; stop and raise it to a human, do not retry")
    grid = next((v["gridData"] for k, v in state.items()
                 if k.startswith("$_") and isinstance(v, dict) and "gridData" in v), None)
    if grid is None:
        raise RuntimeError("no gridData module in __INITIAL_STATE__")
    urls = {}
    for block in re.findall(r'<script[^>]*application/ld\+json[^>]*>(.*?)</script>', html, re.S):
        d = json.loads(block)
        els = d.get("itemListElement") if d.get("@type") == "ItemList" else None
        if isinstance(els, list):             # it is the integer 0 on an empty page
            for el in els:
                urls[el["url"].rsplit("-", 1)[-1]] = el["url"]
    def money(m):
        v = float((m or {}).get("val") or 0)
        return v or None          # 0.0 means "not set" (common for original price)
    rows = []
    for it in grid.get("data") or []:
        bad = [f for f, ok in (("id", re.fullmatch(r"[0-9a-f]{24}", str(it.get("id") or ""))),
                               ("title", it.get("title")), ("price", money(it.get("price_amount"))))
               if not ok]
        if bad:
            raise RuntimeError(f"listing {it.get('id')!r} is missing {bad}; refusing the page")
        rows.append({
            "id": it["id"],
            "title": redact(it["title"]),
            "brand": it.get("brand"),
            "size": (it.get("size_obj") or {}).get("display_with_size_system") or it.get("size"),
            "price": money(it.get("price_amount")),
            "original_price": money(it.get("original_price_amount")),
            "currency": (it.get("price_amount") or {}).get("currency_code"),
            "department": (it.get("department") or {}).get("display"),
            "category": it.get("category"),
            "status": (it.get("inventory") or {}).get("status"),
            "url": urls.get(it["id"], f"{BASE}/listing/{it['id']}"),
        })
    more = grid.get("more") or {}
    return rows, more.get("total"), bool(more.get("is_next_max_id_present"))

def query_items(u):
    return sorted(urllib.parse.parse_qsl(urllib.parse.urlparse(u).query, keep_blank_values=True))

def scrape(path, filters=None, max_pages=3, pause_s=2.0):
    """Returns (rows, status). status is "complete", "truncated_max_pages" (more pages exist)
    or "truncated_site_cap" (Poshmark stopped paging at its ~5000-row cap; treat as a hard stop)."""
    seen, out = set(), []
    for page in range(1, max_pages + 1):
        url = browse_url(path, filters, page)
        final, html = fetch(url)
        fp = urllib.parse.urlparse(final)
        if fp.netloc != "poshmark.com" or not fp.path.startswith(("/brand/", "/category/")):
            raise RuntimeError(f"redirected away from the listings page: {final}")
        if query_items(final) != query_items(url):
            raise RuntimeError(f"server changed the query: asked {url}, got {final}")
        rows, total, has_next = parse_page(html)
        new = [r for r in rows if r["id"] not in seen]
        seen.update(r["id"] for r in new)
        out.extend(new)
        print(f"page {page}: {len(rows)} rows, {len(new)} new, total={total}, next={has_next}")
        if not has_next:
            return out, ("truncated_site_cap" if (total or 0) >= SITE_CAP_TOTAL else "complete")
        if not rows:
            raise RuntimeError(f"page {page} has 0 rows but has_next is true: {final}")
        if page == max_pages:
            return out, "truncated_max_pages"
        time.sleep(pause_s)

listings, status = scrape("brand/Levi's-Women-Jeans",
                          {"sort_by": "added_desc", "size[]": "28", "price[]": "25-50", "availability": "available"},
                          max_pages=3)
print(len(listings), "listings, status:", status)
for r in listings[:3]:
    print(json.dumps(r, ensure_ascii=False))

empty, status = scrape("brand/Zzqxnotabrand-Women-Jeans")
print(len(empty), "listings for an unknown brand, status:", status)
```

Real output (trimmed):

```
page 1: 48 rows, 48 new, total=5000, next=True
page 2: 48 rows, 48 new, total=5000, next=True
page 3: 48 rows, 48 new, total=5000, next=True
144 listings, status: truncated_max_pages
{"id": "6add5dea1921253821f65fea", "title": "Levi's Ribcage Straight Ankle Jeans Distressed Blue Waist 28", "brand": "Levi's", "size": "US 28", "price": 28.0, "original_price": 110.0, "currency": "USD", "department": "Women", "category": "Denim", "status": "available", "url": "https://poshmark.com/listing/Levis-Ribcage-Straight-Ankle-Jeans-Distressed-Blue-Waist-28-6add5dea1921253821f65fea"}
...
page 1: 0 rows, 0 new, total=0, next=False
0 listings for an unknown brand, status: complete
```

`truncated_max_pages` is expected here: `max_pages=3` stopped the run while `has_next` was still true.

On an empty result page the JSON-LD `ItemList.itemListElement` is the integer `0`, not a list; `parse_page` checks for that.

A row with an original price set (from `brand/lululemon_athletica-Women-Pants_&_Jumpsuits`, where 35 of 96 rows had one):

```
{"id": "6ad25d00571a7e8a93781a71", "title": "Lululemon Align High Rise Leggings 28” - Creamy Mint", "brand": "lululemon athletica", "size": "US 6", "price": 35.0, "original_price": 98.0, "currency": "USD", "department": "Women", "category": "Pants", "status": "available", "url": "https://poshmark.com/listing/Lululemon-Align-High-Rise-Leggings-28-Creamy-Mint-6ad25d00571a7e8a93781a71"}
```

### 2. Same data from a browser tab (when you are already driving one)

The hydrated page keeps `window.__INITIAL_STATE__`, so the browser path reads the same JSON instead of scraping tiles. The JSON-LD blocks are not in the live DOM after hydration (0 `ld+json` scripts in a fresh cloud browser, while a same-origin refetch of the HTML had them), so canonical URLs come from the tile links `a[href^="/listing/"]`. The code waits until every listing ID has a rendered tile link.

```python
import json, time, urllib.parse

EXTRACT = r"""(() => {
  if (document.querySelector('#px-captcha, iframe[src*="captcha"]')) return {error: 'challenge on page'};
  if (document.readyState !== 'complete') return {pending: 'readyState ' + document.readyState};
  const s = window.__INITIAL_STATE__;
  if (!s) return {pending: 'no __INITIAL_STATE__ yet'};
  if (s.showCaptcha) return {error: 'captcha'};
  const mod = Object.keys(s).find(k => k.startsWith('$_') && s[k] && s[k].gridData);
  if (!mod) return {pending: 'no gridData yet'};
  const g = s[mod].gridData, data = g.data || [], urls = {};
  // Canonical URLs come from the rendered tile links (the JSON-LD ItemList is gone from the live DOM).
  document.querySelectorAll('a[href^="/listing/"]').forEach(a => {
    const h = a.getAttribute('href').split('?')[0];
    urls[h.split('-').pop()] = location.origin + h;
  });
  const missing = data.filter(it => !urls[it.id]).length;
  if (missing) return {pending: `${missing}/${data.length} tiles not rendered yet`};
  const money = m => { const v = parseFloat((m || {}).val || '0'); return v || null; };
  const bad = data.find(it => !/^[0-9a-f]{24}$/.test(it.id || '') || !it.title || !money(it.price_amount));
  if (bad) return {error: `listing ${bad.id} is missing id/title/price`};
  // Titles are seller-written free text: mask any email address or phone number.
  const redact = t => t.replace(/[\w.+-]+@[\w-]+\.[\w.-]+/g, '[email]')
    .replace(/(?<!\d)(?:\+?1[\s.-]?)?\(?\d{3}\)?[\s.-]?\d{3}[\s.-]?\d{4}(?!\d)/g, '[phone]');
  return {
    total: (g.more || {}).total, has_next: !!(g.more || {}).is_next_max_id_present,
    rows: data.map(it => ({
      id: it.id, title: redact(it.title), brand: it.brand,
      size: (it.size_obj || {}).display_with_size_system || it.size,
      price: money(it.price_amount), original_price: money(it.original_price_amount),
      url: urls[it.id],
    })),
  };
})()"""

def same_url(a, b):
    """Same host, path and query (order and %-encoding ignored), so page N is never read as page N-1."""
    pa, pb = urllib.parse.urlparse(a), urllib.parse.urlparse(b)
    q = lambda p: sorted(urllib.parse.parse_qsl(p.query, keep_blank_values=True))
    return (pa.netloc, urllib.parse.unquote(pa.path), q(pa)) == (pb.netloc, urllib.parse.unquote(pb.path), q(pb))

def open_page(url, first=False, settle_s=20):
    new_tab(url) if first else goto_url(url)
    wait_for_load()
    deadline, href, data = time.monotonic() + settle_s, None, None
    while time.monotonic() < deadline:       # one deadline for the URL, state and tiles to settle
        href = js("location.href")
        if same_url(href or "", url):
            data = js(EXTRACT)
            if data and data.get("error"):
                raise RuntimeError(f"extraction failed on {href}: {data['error']}")
            if data and "rows" in data:
                return data
        time.sleep(0.5)
    raise RuntimeError(f"page did not settle on {url}: href={href} state={data and data.get('pending')}")

MAX_PAGES, SITE_CAP_TOTAL = 3, 5000
base = "https://poshmark.com/category/Men-Shoes?" + urllib.parse.urlencode(
    {"brand[]": "Nike", "sort_by": "added_desc"})
results, seen, status = [], set(), None
for page in range(1, MAX_PAGES + 1):
    url = base + (f"&max_id={page}" if page > 1 else "")
    data = open_page(url, first=(page == 1))
    new = [r for r in data["rows"] if r["id"] not in seen]
    seen.update(r["id"] for r in new)
    results.extend(new)
    print(f"page {page}: {len(data['rows'])} rows, {len(new)} new, next={data['has_next']}")
    if not data["has_next"]:
        status = "truncated_site_cap" if (data["total"] or 0) >= SITE_CAP_TOTAL else "complete"
        break
    if not data["rows"]:
        raise RuntimeError(f"page {page} has 0 rows but has_next is true")
    if page == MAX_PAGES:
        status = "truncated_max_pages"
        break
    time.sleep(2)
print(len(results), "listings, status:", status)
print(json.dumps(results[:2], indent=1, ensure_ascii=False))
```

Real output (trimmed):

```
page 1: 48 rows, 48 new, next=True
page 2: 48 rows, 48 new, next=True
page 3: 48 rows, 48 new, next=True
144 listings, status: truncated_max_pages
[
 {
  "id": "6add5f6d06c773fcadec8423",
  "title": "Size 12 - Nike Zoom KD 12 EYBL Peach Jam",
  "brand": "Nike",
  "size": "US 12",
  "price": 187,
  "original_price": null,
  "url": "https://poshmark.com/listing/Size-12-Nike-Zoom-KD-12-EYBL-Peach-Jam-6add5f6d06c773fcadec8423"
 },
 ...
```

## URL patterns and filters (all observed)

| Pattern | Result |
|---|---|
| `/brand/<Brand>` (e.g. `/brand/Patagonia`) | 200, 48 rows, all departments |
| `/brand/<Brand>-<Dept>-<Category>` (e.g. `/brand/Levi's-Women-Jeans`) | 200, 48 rows, module `$_brand` |
| `/category/<Dept>-<Category>` (e.g. `/category/Women-Jeans`, `/category/Men-Shoes`) | 200, 48 rows, module `$_category` |
| `?max_id=N` | page N (what the site's Next button generates) |
| `?sort_by=` | values seen in the sort radio inputs: `best_match`, `added_desc` (Just In), `price_desc`, `price_asc`, `price_drop`, `like_count`, `relevance_v2` |
| `?size[]=28` | size filter, repeatable |
| `?price[]=25-50` | price bucket from the Price filter ("$25 - $50"); click other buckets in the UI and copy the URL rather than guessing their format |
| `?brand[]=Nike` | brand filter on `/category/...` pages |
| `?availability=available` | added by the UI; rows had `inventory.status == "available"` |
| Unknown brand slug, or a filter with no matches (`/brand/Patagonia?size[]=ZZ999`) | 200 with 0 rows, `total: 0`, and JSON-LD `itemListElement: 0` (not a 404), so check for empty results |
| `/brand/Lululemon_Athletica-Women-Pants` | 301 to `/brand/lululemon_athletica-Women-Pants_&_Jumpsuits` (urllib follows it) |
| `/listing/<id>` | 301 to the canonical `/listing/<Title-Slug>-<id>` URL |

## Fields

| Output field | Source in `gridData.data[i]` | Notes |
|---|---|---|
| `id` | `id` | 24-hex listing ID, also the last segment of the listing URL |
| `title` | `title` | seller-written, may contain Unicode quotes; email addresses and phone numbers are replaced with `[email]` / `[phone]` |
| `brand` | `brand` | display name, e.g. `"lululemon athletica"` |
| `size` | `size_obj.display_with_size_system`, fallback `size` | e.g. `"US 28"`, or multi-system like `"US 11 / UK 10 / EUR 45 / cm 29"`; `size` alone is `"28"`, and can be `"Various"` |
| `price` | `price_amount.val` (string) | also integer `price` |
| `original_price` | `original_price_amount.val` | `"0.0"` when the seller left it blank, mapped to `None` |
| `currency` | `price_amount.currency_code` | `"USD"` on poshmark.com |
| `department` | `department.display` | `"Women"`, `"Men"`, ... |
| `category` | `category` | e.g. `"Denim"`, `"Pants"` |
| `status` | `inventory.status` | `"available"` |
| `url` | HTTP: JSON-LD `ItemList.itemListElement[].url`, matched by ID. Browser: tile link `a[href^="/listing/"]` | 48 of 48 matched on every page tested |
| (page) `total` | `gridData.more.total` | capped at 5000; `0` on an empty page |
| (page) `has_next` | `gridData.more.is_next_max_id_present` | false on the last page |

Each listing object also carries seller fields (`creator_username`, `creator_full_name`, `creator_picture_url`, ...). The code above deliberately does not copy them; leave them out unless you have a reason and a basis to process personal data.

## Gotchas

- `size[]`, `price[]`, `brand[]` need the brackets URL-encoded (`size%5B%5D=28`); `urllib.parse.urlencode` does this. Brand slugs with an apostrophe (`Levi's`) work encoded as `%27`, and `&` in category slugs (`Pants_&_Jumpsuits`) works as `%26`.
- Default sort (relevance) is not stable between requests: two fetches of page 2 of `/category/Women-Jeans` (one via `max_id=2`, one via the `next_max_id` token from page 1) returned 48 rows each with zero IDs in common. Use `sort_by=added_desc` for paging. New listings arrive constantly (page 2 of Levi's women's jeans covered about 40 minutes), so a listing can shift onto the next page; the code de-duplicates by `id`.
- `more.next_max_id` is an opaque `ENC_...` token; plain `max_id=N` worked for every page tested and is what the UI uses.
- `more.total` says 5000 even for narrow filters (`size[]=28&price[]=25-50`), so it means "5000 or more", not an exact count. Page 105 is the last page (`has_next: false`); see the Quick summary for what pages past it return.
- No consent banner or login wall appeared on browse pages in the cloud browser. `state.ipCountry` was `"US"` and `state.domain` was `"us"`; Poshmark has other country sites, so check `currency` if you run from elsewhere.
- In the browser, tile counts via `[data-et-name="listing"]` returned 144 for 48 listings, and `a[href^="/listing/"]` returned 96 (two links per tile). Read `__INITIAL_STATE__` for the rows and use the links only to look up URLs by ID.
- In the browser, the JSON-LD `<script type="application/ld+json">` blocks are absent from the live DOM after hydration, so reading them right after load is a race (a first run returned fallback `/listing/<id>` URLs). Block 2 uses tile links and waits until every ID is matched.
- Size filter buttons (`#size-28`) are hidden until the Size dropdown is opened, and the choice only applies after clicking Apply. Building the URL directly is simpler.
- Pacing: 2 s between pages was used throughout with no 429 or captcha. Keep at least that. On any 403 or `showCaptcha: true` the code raises; stop and hand it to a human, do not retry or change the request to get past it. `fetch` retries only 429/5xx/network errors, all under one 120 s deadline per URL, with no sleep after the last attempt.
- Both blocks refuse a page if the browser or server ended up on a different path or query than requested, or if any listing lacks a 24-hex `id`, a `title` or a nonzero `price`.

## What does not work

- `/search`, `/api/...` and `/listings` are disallowed by robots.txt for `User-agent: *`, so they were not requested.
- There is no keyword parameter used on browse pages; to approximate keyword search, pick the closest brand/category, add filters, and match on `title` locally.
- Going past the result cap: with `sort_by=added_desc`, pages 106 and later return 0 rows. The cap is site-enforced; do not reshape or split the query to get around it. Report the result as truncated.
- Sold items: the UI's `availability=available` and every row observed were available listings; sold-listing scraping was not tested.
