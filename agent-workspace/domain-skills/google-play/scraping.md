# Google Play — App Details and Search Results

Field-tested against play.google.com on 2026-10-23 using plain HTTP (Python urllib) and a Browser Use cloud browser.

## Quick summary

- **App detail pages** (`/store/apps/details?id=<package>`) return the full record over plain HTTP: one JSON-LD `SoftwareApplication` block plus `AF_initDataCallback` JSON blobs. Every field people ask for (title, developer, rating, ratings count, installs, price, category, last updated, version, content rating) is in there. No browser needed.
- **Search pages** (`/store/search?q=...&c=apps`) embed the result list in the `ds:4` callback. Most queries return a single page of **18-30 apps** (observed: pdf scanner 18, spotify and roblox and candy crush 1 top result + 18-19, budget app 21, photo editor 23, music player 23, fitness tracker 25, chess 29, vpn/weather/calculator 30).
- **Only some queries have more pages.** `minecraft` carried a continuation token; following it through the `qnKhOb` batchexecute RPC returned 50 apps per call up to a hard stop at **250** (1 top result + 22 + 50 + 50 + 50 + 50 + 27). The other 12 queries tested had no token.
- Search cards lack ratings count, last updated and version, so the reliable pattern is: search -> package names -> one detail request per app.
- **Developer pages** (`/store/apps/developer?id=<exact name>`) show up to 20 apps on first load, then page through the same token RPC: Google LLC 90 apps, Microsoft Corporation 72, Rovio Entertainment Oy 16, Mojang 3. A name that does not match exactly returns HTTP 404.
- About 250 HTTP requests plus about 20 browser page loads over the day at 0.5-1 s spacing: no 429, no CAPTCHA, no consent interstitial.
- The plain-HTTP code sends a truthful client User-Agent (`browser-harness-skill/1.0 (Python urllib)`). Detail, search, developer pages and the `qnKhOb` RPC all answered it the same as a Chrome UA on 2026-10-23, so no browser UA is needed.

## What works

### 1. Details for package names, and for every result of a search (plain HTTP only)

Runs as a single script. `play_app()` handles one package name; `play_search()` returns `(results, complete)`, following continuation tokens when the query has them. Every request retries 429/5xx and network errors with backoff (honouring `Retry-After` as seconds or an HTTP date) and raises if it still fails, so a short list is never a silent failure. `complete` is `False` when `max_results` or `max_pages` stopped the walk while Play still offered another page.

```python
import re, json, time, email.utils, urllib.parse, urllib.request, urllib.error

H = {"User-Agent": "browser-harness-skill/1.0 (Python urllib)", "Accept-Language": "en-US,en;q=0.9"}
HL, GL = "en_US", "US"          # language and store country (country sets currency and availability)
RETRY_CODES = {429, 500, 502, 503, 504}

def _retry_after(value):
    """Retry-After as seconds, or as an HTTP date; None if absent or unparseable."""
    if not value:
        return None
    if value.strip().isdigit():
        return int(value.strip())
    try:
        return max(0.0, email.utils.parsedate_to_datetime(value).timestamp() - time.time())
    except (TypeError, ValueError):
        return None

def fetch(url, data=None, headers=None, tries=4):
    """GET (or POST when data is given). Retries 429/5xx and network errors with backoff, then raises.
    Other HTTP errors (404 included) raise urllib.error.HTTPError immediately."""
    for i in range(tries):
        req = urllib.request.Request(url, data=data, headers={**H, **(headers or {})})
        try:
            with urllib.request.urlopen(req, timeout=20) as r:
                return r.read().decode()
        except urllib.error.HTTPError as e:
            if e.code not in RETRY_CODES or i == tries - 1:
                raise
            wait = _retry_after(e.headers.get("Retry-After")) or 2 ** (i + 1)
        except (urllib.error.URLError, TimeoutError, ConnectionError):
            if i == tries - 1:
                raise
            wait = 2 ** (i + 1)
        time.sleep(min(wait, 120))

def callbacks(html):
    """Yield every AF_initDataCallback payload embedded in a Play page."""
    for m in re.finditer(r"AF_initDataCallback\(\{key: '(ds:\d+)', hash: '\d+', data:(.*?), sideChannel: \{\}\}\);</script>", html, re.S):
        try:
            yield m.group(1), json.loads(m.group(2))
        except ValueError:
            pass

def g(o, *path):
    """Safe nested index: returns None instead of raising."""
    for p in path:
        try:
            o = o[p]
        except (IndexError, KeyError, TypeError):
            return None
    return o

def num(x):
    return x if isinstance(x, (int, float)) and not isinstance(x, bool) else None

def text(x):
    return x if isinstance(x, str) else None

def offer_fields(o):
    """[micros, currency, text] -> (price, currency, price_text). Free apps have micros 0 and an empty text."""
    o = o if isinstance(o, list) else []
    micros = num(g(o, 0))
    return ((micros / 1e6) if micros is not None else None, text(g(o, 1)),
            text(g(o, 2)) or ("Free" if micros == 0 else None))

def ld_app(html, app_id):
    """The JSON-LD SoftwareApplication block for this package, or {} (never another app's block)."""
    for m in re.finditer(r'<script type="application/ld\+json"[^>]*>(.*?)</script>', html, re.S):
        try:
            d = json.loads(m.group(1))
        except ValueError:
            continue
        for item in (d if isinstance(d, list) else [d]):
            if not isinstance(item, dict) or "applicationCategory" not in item:
                continue
            ids = urllib.parse.parse_qs(urllib.parse.urlparse(text(item.get("url")) or "").query).get("id")
            if ids in (None, [app_id]):
                return item
    return {}

def ld_author(ld):
    a = ld.get("author")
    a = a[0] if isinstance(a, list) and a else a
    return text(a.get("name")) if isinstance(a, dict) else text(a)

def play_app(app_id):
    """Full detail record for one package name, or None if the app is not in this store (HTTP 404)."""
    url = "https://play.google.com/store/apps/details?" + urllib.parse.urlencode({"id": app_id, "hl": HL, "gl": GL})
    try:
        html = fetch(url)
    except urllib.error.HTTPError as e:
        if e.code == 404:
            return None
        raise
    a = next((g(d, 1, 2) for _, d in callbacks(html) if g(d, 1, 2, 77, 0) == app_id), None)
    ld = ld_app(html, app_id)
    if a is None and not ld:
        return None          # not an app page (removed app, region-locked, or an error page)
    price, currency, price_text = offer_fields(g(a, 57, 0, 0, 0, 0, 1, 0))
    return {
        "app_id": app_id,
        "title": text(g(a, 0, 0)) or text(ld.get("name")),
        "developer": text(g(a, 68, 0)) or ld_author(ld),
        "developer_id": text(g(a, 68, 2)),
        "rating": num(g(a, 51, 0, 1)),
        "ratings_count": num(g(a, 51, 2, 1)),
        "installs": text(g(a, 13, 0)),
        "min_installs": num(g(a, 13, 1)),
        "installs_exact": num(g(a, 13, 2)),
        "price": price,
        "currency": currency,
        "price_text": price_text,
        "category": text(g(a, 79, 0, 0, 0)),
        "category_id": text(g(a, 79, 0, 0, 2)) or text(ld.get("applicationCategory")),
        "updated": text(g(a, 145, 0, 0)),
        "updated_ts": num(g(a, 145, 0, 1, 0)),
        "version": text(g(a, 140, 0, 0, 0)) or ("Varies with device" if a else None),
        "content_rating": text(g(a, 9, 0)) or text(ld.get("contentRating")),
        "released": text(g(a, 10, 0)),
        "contains_ads": bool(g(a, 48, 0)),
        "in_app_purchases": text(g(a, 19, 0)),
        "url": url,
    }

RPC_FIELDS = [96, 27, 4, 8, 57, 30, 110, 79, 11, 16, 49, 1, 3, 9, 12, 104, 55, 56, 51, 10, 34, 77]

def next_page(token):
    """One qnKhOb continuation call -> (cards, next_token or None). Raises if the answer is not a result page."""
    inner = json.dumps([[None, [[10, [10, 50]], True, None, RPC_FIELDS], None, token]])
    body = urllib.parse.urlencode({"f.req": json.dumps([[["qnKhOb", inner, None, "generic"]]])}).encode()
    txt = fetch("https://play.google.com/_/PlayStoreUi/data/batchexecute?" + urllib.parse.urlencode({"rpcids": "qnKhOb", "hl": HL, "gl": GL}),
                data=body, headers={"Content-Type": "application/x-www-form-urlencoded;charset=UTF-8"})
    line = next((l for l in txt.split("\n") if l.startswith('[["wrb.fr"')), None)
    raw = g(json.loads(line), 0, 2) if line else None
    if not isinstance(raw, str):
        raise RuntimeError("qnKhOb: no result payload in response")
    payload = json.loads(raw)
    cards = g(payload, 0, 0, 0)
    if not isinstance(cards, list):
        raise RuntimeError("qnKhOb: no card list in payload")
    tok = g(payload, 0, 0, 7, 1)
    return cards, (tok if isinstance(tok, str) and tok else None)

def _search_cards(node, out):
    """Collect first-page result cards: lists shaped [[app_id, 7], icon, shots, title, [rating_text, rating], category, ...]."""
    if isinstance(node, list):
        if (len(node) > 15 and isinstance(node[0], list) and len(node[0]) == 2 and node[0][1] == 7
                and isinstance(node[0][0], str) and isinstance(node[3], str)):
            out.append(node)
            return
        for v in node:
            _search_cards(v, out)

def play_search(query, max_results=100, max_pages=10, pause=1.0):
    """(results, complete). results: ordered dicts for a search query; complete is False if a cap cut it short."""
    url = "https://play.google.com/store/search?" + urllib.parse.urlencode({"q": query, "c": "apps", "hl": HL, "gl": GL})
    data = dict(callbacks(fetch(url))).get("ds:4")
    if data is None:
        raise RuntimeError(f"search page for {query!r} has no ds:4 block; Play markup changed")
    results, seen, token = [], set(), None
    def add(rec):
        if rec["app_id"] is None or rec["app_id"] not in seen:     # dedupe by package name, keep id-less rows
            seen.add(rec["app_id"]); results.append(rec)
    for sec in g(data, 0, 1) or []:
        name = text(g(sec, 3, 1, 0, 0)) or ""
        featured = g(sec, 23)            # big "top result" card on brand queries
        if featured:
            m = re.search(r'details\?id=([\w.]+)', json.dumps(featured))
            if m:
                add({"app_id": m.group(1), "title": text(g(featured, 16, 2, 0, 0)), "source": "top_result"})
        if name.startswith("search_results_cluster"):   # skip "similar_apps" and related-search sections
            cards = []
            _search_cards(g(sec, 22), cards)
            for c in cards:
                add({"app_id": c[0][0], "title": c[3], "developer": text(g(c, 14)), "rating": num(g(c, 4, 1)),
                     "category": text(g(c, 5)), "installs": text(g(c, 15)), "price_text": offer_fields(g(c, 8, 1, 0))[2],
                     "source": "page1"})
            tok = g(sec, 22, 1, 3, 1)
            token = tok if isinstance(tok, str) and tok else token
    page = 1
    while token and len(results) < max_results and page < max_pages:
        time.sleep(pause)
        page += 1
        cards, token = next_page(token)
        for c in cards:
            add({"app_id": text(g(c, 12, 0)), "title": text(g(c, 2)), "developer": text(g(c, 4, 0, 0, 0)),
                 "rating": num(g(c, 6, 0, 2, 1, 1)), "source": f"page{page}"})
    complete = token is None and len(results) <= max_results
    if not complete:
        print(f"play_search({query!r}): stopped by a cap (max_results={max_results}, max_pages={max_pages}); more results exist")
    return results[:max_results], complete

# --- 1) details for known package names ---
for app_id in ["com.spotify.music", "com.mojang.minecraftpe", "org.videolan.vlc", "com.example.does.not.exist"]:
    rec = play_app(app_id)
    print(json.dumps(rec) if rec else f"{app_id}: not found (404)")
    time.sleep(1)

# --- 2) a search query's results, then full details for each ---
hits, complete = play_search("minecraft", max_results=80)
print("search hits:", len(hits), "complete:", complete, {s: sum(h["source"] == s for h in hits) for s in sorted({h["source"] for h in hits})})
print("first card:", json.dumps(hits[1]))
rows = []
for h in hits[:25]:
    rec = play_app(h["app_id"]) if h["app_id"] else None
    if rec:
        rows.append(rec)
    time.sleep(1)
print("detailed:", len(rows))
for r in rows[:5]:
    print(r["app_id"], "|", r["title"], "|", r["developer"], "|", r["rating"], r["ratings_count"], "|",
          r["installs"], "|", r["price_text"], "|", r["category"], "|", r["updated"], "|", r["version"], "|", r["content_rating"])

time.sleep(1)
w, complete = play_search("weather")
print("weather:", len(w), "complete:", complete)
time.sleep(1)
m, complete = play_search("minecraft", max_results=1000)
print("minecraft, no cap hit:", len(m), "complete:", complete)
```

Output (from the run on 2026-10-23):

```text
{"app_id": "com.spotify.music", "title": "Spotify: Music and Podcasts", "developer": "Spotify AB", "developer_id": "6523013212965523194", "rating": 4.3489933, "ratings_count": 36528080, "installs": "1,000,000,000+", "min_installs": 1000000000, "installs_exact": 3171683950, "price": 0.0, "currency": "USD", "price_text": "Free", "category": "Music & Audio", "category_id": "MUSIC_AND_AUDIO", "updated": "Oct 23, 2026", "updated_ts": 1792759042, "version": "Varies with device", "content_rating": "Teen", "released": "May 27, 2014", "contains_ads": true, "in_app_purchases": "$4.99 - $203.88 per item", "url": "https://play.google.com/store/apps/details?id=com.spotify.music&hl=en_US&gl=US"}
{"app_id": "com.mojang.minecraftpe", "title": "Minecraft: Dream it, Build it!", "developer": "Mojang", "developer_id": "4772240228547998649", "rating": 4.303256, "ratings_count": 5866785, "installs": "50,000,000+", "min_installs": 50000000, "installs_exact": 99618770, "price": 6.99, "currency": "USD", "price_text": "$6.99", "category": "Arcade", "category_id": "GAME_ARCADE", "updated": "Sep 25, 2026", "updated_ts": 1790357188, "version": "1.26.52.3", "content_rating": "Everyone 10+", "released": "Aug 15, 2011", "contains_ads": false, "in_app_purchases": "$0.99 - $49.99 per item", "url": "https://play.google.com/store/apps/details?id=com.mojang.minecraftpe&hl=en_US&gl=US"}
{"app_id": "org.videolan.vlc", "title": "VLC for Android", "developer": "Videolabs", "developer_id": "6364851808230428105", "rating": 4.042392, "ratings_count": 2009174, "installs": "500,000,000+", "min_installs": 500000000, "installs_exact": 515866848, "price": 0.0, "currency": "USD", "price_text": "Free", "category": "Video Players & Editors", "category_id": "VIDEO_PLAYERS", "updated": "Oct 14, 2026", "updated_ts": 1791980045, "version": "Varies with device", "content_rating": "Everyone", "released": "Feb 4, 2015", "contains_ads": false, "in_app_purchases": null, "url": "https://play.google.com/store/apps/details?id=org.videolan.vlc&hl=en_US&gl=US"}
com.example.does.not.exist: not found (404)
play_search('minecraft'): stopped by a cap (max_results=80, max_pages=10); more results exist
search hits: 80 complete: False {'page1': 22, 'page2': 50, 'page3': 7, 'top_result': 1}
first card: {"app_id": "com.mojang.minecraftedu", "title": "Minecraft Education", "developer": "Mojang", "rating": 4.132686, "category": "Education", "installs": "50,000,000+", "price_text": "Free", "source": "page1"}
detailed: 25
com.mojang.minecraftpe | Minecraft: Dream it, Build it! | Mojang | 4.303237 5866785 | 50,000,000+ | $6.99 | Arcade | Sep 25, 2026 | 1.26.52.3 | Everyone 10+
com.mojang.minecraftedu | Minecraft Education | Mojang | 4.132686 152610 | 50,000,000+ | Free | Education | Jul 30, 2026 | 1.26.32.0 | Everyone 10+
com.mojang.minecraftedu_preview | Minecraft Education Preview | Mojang | 3.6 22549 | 10,000,000+ | Free | Education | Jun 25, 2026 | 1.26.30.0 | Everyone 10+
com.mcmodshub.app | MC Bedrock Mods for Minecraft | MCModsHub | 4.642857 21831 | 1,000,000+ | Free | Casual | Oct 19, 2026 | 1.13.0 | Everyone
com.fungames.blockcraft | Block Craft 3D：Building Game | Wildlife Studios | 4.486032 3053490 | 500,000,000+ | Free | Arcade | Oct 22, 2026 | 4.9.1 | Everyone
weather: 30 complete: True
minecraft, no cap hit: 250 complete: True
```

### 2. Same core fields from the rendered page (cloud browser fallback)

Use this if plain HTTP starts returning something other than an app page. JSON-LD carries title, developer, rating, ratings count, price, currency, category id and content rating. Installs and last updated come from the visible page text. `open_play()` waits until the tab's URL has the requested path and `id`/`q` before reading anything, so it never parses the previous page. Harness timeouts (`TimeoutError` from the daemon, or the `Runtime.evaluate timed out` RuntimeError from `js()`) are retried. Nothing here brings the tab to the foreground.

```python
import json, time, urllib.parse

def _retry(fn, *args, tries=3):
    """Call a harness helper, retrying only its timeouts: TimeoutError from the daemon IPC,
    or the 'Runtime.evaluate timed out' RuntimeError that js() raises. Other errors propagate."""
    for i in range(tries):
        try:
            return fn(*args)
        except (TimeoutError, RuntimeError) as e:
            if i == tries - 1 or not (isinstance(e, TimeoutError) or "timed out" in str(e)):
                raise
            time.sleep(2)

def open_play(path, params, ready_expr, timeout=30):
    """Navigate, then wait until the loaded document is THIS url (same path, same id/q) and ready_expr is true."""
    url = "https://play.google.com" + path + "?" + urllib.parse.urlencode(params)
    _retry(goto_url, url)
    check = f"""(() => {{
      const u = new URL(location.href), want = {json.dumps(params)};
      if (u.pathname !== {json.dumps(path)}) return false;
      for (const k of ['id', 'q']) if (k in want && u.searchParams.get(k) !== want[k]) return false;
      return document.readyState === 'complete' && Boolean({ready_expr});
    }})()"""
    deadline = time.time() + timeout
    while time.time() < deadline:
        if _retry(js, check):
            return url
        time.sleep(0.5)
    raise TimeoutError(f"{url} did not finish loading within {timeout}s")

def browser_app(app_id):
    """Same core fields from the rendered page: JSON-LD plus the labelled 'About this app' facts."""
    open_play("/store/apps/details", {"id": app_id, "hl": "en_US", "gl": "US"},
              "document.querySelector('script[type=\"application/ld+json\"]') || document.title.includes('Not Found')")
    return _retry(js, """(() => {
      const want = %s;
      let ld = {};
      for (const s of document.querySelectorAll('script[type="application/ld+json"]')) {
        try {
          const d = JSON.parse(s.textContent);
          for (const it of (Array.isArray(d) ? d : [d])) {
            const id = it && it.url ? new URL(it.url, location.href).searchParams.get('id') : want;
            if (it && it.applicationCategory && id === want) { ld = it; break; }
          }
        } catch (e) {}
      }
      const first = v => Array.isArray(v) ? v[0] : v;
      const n = v => (v === undefined || v === null || v === '' || isNaN(Number(v))) ? null : Number(v);
      const author = first(ld.author), offer = first(ld.offers) || {}, agg = ld.aggregateRating || {};
      const txt = document.body.innerText;
      const pick = re => { const m = txt.match(re); return m ? m[1].trim() : null; };
      return {
        app_id: new URL(location.href).searchParams.get('id'),
        title: ld.name || null,
        developer: typeof author === 'string' ? author : (author && author.name) || null,
        rating: n(agg.ratingValue),
        ratings_count: n(agg.ratingCount),
        price: n(offer.price),
        currency: offer.priceCurrency || null,
        category_id: ld.applicationCategory || null,
        content_rating: ld.contentRating || null,
        installs_badge: pick(/\\n([\\d.,]+[KMB]?\\+)\\n\\s*Downloads/),
        updated: pick(/Updated on\\n([^\\n]+)/),
        not_found: document.title.includes('Not Found') || !ld.name,
      };
    })()""" % json.dumps(app_id))

new_tab("about:blank")
for app_id in ["com.spotify.music", "com.mojang.minecraftpe", "com.example.does.not.exist"]:
    print(json.dumps(browser_app(app_id)))

# search results in the rendered page: unique package names in DOM order (every app link on the page)
IDS_JS = "[...new Set([...document.querySelectorAll('a[href*=\"/store/apps/details?id=\"]')].map(a => new URL(a.href).searchParams.get('id')))]"
open_play("/store/search", {"q": "weather", "c": "apps", "hl": "en_US", "gl": "US"},
          "document.querySelector('a[href*=\"/store/apps/details?id=\"]')")
ids, prev, settled = [], None, False
for _ in range(10):              # the rendered list grows for a few seconds; stop once two reads agree
    ids = _retry(js, IDS_JS)
    if ids == prev:
        settled = True
        break
    prev = ids
    time.sleep(1.5)
print(len(ids), "settled:", settled, ids[:10])
```

Output (from the run on 2026-10-23):

```text
{"app_id": "com.spotify.music", "title": "Spotify: Music and Podcasts", "developer": "Spotify AB", "rating": 4.348993301391602, "ratings_count": 36528080, "price": 0, "currency": "USD", "category_id": "MUSIC_AND_AUDIO", "content_rating": "Teen", "installs_badge": "1B+", "updated": "Oct 23, 2026", "not_found": false}
{"app_id": "com.mojang.minecraftpe", "title": "Minecraft: Dream it, Build it!", "developer": "Mojang", "rating": 4.303256034851074, "ratings_count": 5866742, "price": 6.99, "currency": "USD", "category_id": "GAME_ARCADE", "content_rating": "Everyone 10+", "installs_badge": "50M+", "updated": "Sep 25, 2026", "not_found": false}
{"app_id": "com.example.does.not.exist", "title": null, "developer": null, "rating": null, "ratings_count": null, "price": null, "currency": null, "category_id": null, "content_rating": null, "installs_badge": null, "updated": null, "not_found": true}
25 settled: True ['com.weather.Weather', 'com.accuweather.android', 'aplicacion.tiempo', 'com.google.android.apps.weather', 'de.wetteronline.wetterapp', 'com.inmobi.weather', 'com.wunderground.android.weather', 'com.weawow', 'com.acmeaom.android.myradar', 'com.pelmorex.WeatherEyeAndroid']
```

The rendered `weather` list settled at 25 here while the HTTP path returned 30 for the same query (see Gotchas).

`browser_app()` returns `app_id`, `title`, `developer`, `rating`, `ratings_count`, `price`, `currency`, `category_id`, `content_rating`, `installs_badge`, `updated`, `not_found`. JSON-LD is only used when its `url` carries the same `id`, so a miss gives `null`s and `not_found: true`, never another app's data.

### 3. A developer's app list, with pagination (plain HTTP plus one POST per extra page)

Self-contained, same retry and cap rules as block 1. `developer_name()` reads the developer name from the app page exactly as Play spells it; `developer_apps()` loads `/store/apps/developer?id=<name>` and follows the same `qnKhOb` continuation token as search. It returns `None` when Play answers 404 (no such developer page), otherwise `(apps, complete)`. Feed the package names into `play_app()` for full records.

```python
import re, json, time, email.utils, urllib.parse, urllib.request, urllib.error

H = {"User-Agent": "browser-harness-skill/1.0 (Python urllib)", "Accept-Language": "en-US,en;q=0.9"}
HL, GL = "en_US", "US"
RETRY_CODES = {429, 500, 502, 503, 504}
RPC_FIELDS = [96, 27, 4, 8, 57, 30, 110, 79, 11, 16, 49, 1, 3, 9, 12, 104, 55, 56, 51, 10, 34, 77]

def _retry_after(value):
    if not value:
        return None
    if value.strip().isdigit():
        return int(value.strip())
    try:
        return max(0.0, email.utils.parsedate_to_datetime(value).timestamp() - time.time())
    except (TypeError, ValueError):
        return None

def fetch(url, data=None, headers=None, tries=4):
    """Retries 429/5xx (Retry-After as seconds or HTTP date) and network errors, then raises."""
    for i in range(tries):
        req = urllib.request.Request(url, data=data, headers={**H, **(headers or {})})
        try:
            with urllib.request.urlopen(req, timeout=20) as r:
                return r.read().decode()
        except urllib.error.HTTPError as e:
            if e.code not in RETRY_CODES or i == tries - 1:
                raise
            wait = _retry_after(e.headers.get("Retry-After")) or 2 ** (i + 1)
        except (urllib.error.URLError, TimeoutError, ConnectionError):
            if i == tries - 1:
                raise
            wait = 2 ** (i + 1)
        time.sleep(min(wait, 120))

def g(o, *path):
    for p in path:
        try:
            o = o[p]
        except (IndexError, KeyError, TypeError):
            return None
    return o

def num(x):
    return x if isinstance(x, (int, float)) and not isinstance(x, bool) else None

def text(x):
    return x if isinstance(x, str) else None

def next_page(token):
    """One qnKhOb continuation call -> (cards, next_token or None). Raises if the answer is not a result page."""
    inner = json.dumps([[None, [[10, [10, 50]], True, None, RPC_FIELDS], None, token]])
    body = urllib.parse.urlencode({"f.req": json.dumps([[["qnKhOb", inner, None, "generic"]]])}).encode()
    txt = fetch("https://play.google.com/_/PlayStoreUi/data/batchexecute?" + urllib.parse.urlencode({"rpcids": "qnKhOb", "hl": HL, "gl": GL}),
                data=body, headers={"Content-Type": "application/x-www-form-urlencoded;charset=UTF-8"})
    line = next((l for l in txt.split("\n") if l.startswith('[["wrb.fr"')), None)
    raw = g(json.loads(line), 0, 2) if line else None
    if not isinstance(raw, str):
        raise RuntimeError("qnKhOb: no result payload in response")
    payload = json.loads(raw)
    cards = g(payload, 0, 0, 0)
    if not isinstance(cards, list):
        raise RuntimeError("qnKhOb: no card list in payload")
    tok = g(payload, 0, 0, 7, 1)
    return cards, (tok if isinstance(tok, str) and tok else None)

def developer_name(app_id):
    """Developer name exactly as Play spells it (JSON-LD author.name of this app's page), or None if the app is not found."""
    url = "https://play.google.com/store/apps/details?" + urllib.parse.urlencode({"id": app_id, "hl": HL, "gl": GL})
    try:
        html = fetch(url)
    except urllib.error.HTTPError as e:
        if e.code == 404:
            return None
        raise
    for m in re.finditer(r'<script type="application/ld\+json"[^>]*>(.*?)</script>', html, re.S):
        try:
            ld = json.loads(m.group(1))
        except ValueError:
            continue
        ids = urllib.parse.parse_qs(urllib.parse.urlparse(text(g(ld, "url")) or "").query).get("id")
        if isinstance(ld, dict) and "applicationCategory" in ld and ids in (None, [app_id]):
            a = ld.get("author")
            a = a[0] if isinstance(a, list) and a else a
            return text(a.get("name")) if isinstance(a, dict) else text(a)
    return None

def developer_apps(name, max_results=500, max_pages=20, pause=1.0):
    """None if Play answers 404 (name does not match a developer page exactly), else (apps, complete).
    complete is False when max_results or max_pages stopped the walk while another page existed."""
    url = "https://play.google.com/store/apps/developer?" + urllib.parse.urlencode({"id": name, "hl": HL, "gl": GL})
    try:
        html = fetch(url)
    except urllib.error.HTTPError as e:
        if e.code == 404:
            return None
        raise
    results, seen, token, found = [], set(), None, False
    def add(app_id, title, rating, page):
        if app_id is None or app_id not in seen:     # dedupe by package name, keep id-less rows
            seen.add(app_id)
            results.append({"app_id": app_id, "title": title, "rating": rating, "source": page})
    def walk(node):   # first-page cards are lists shaped [[app_id, 7], icon, shots, title, [rating_text, rating], ...]
        if isinstance(node, list):
            if (len(node) > 15 and isinstance(node[0], list) and len(node[0]) == 2 and node[0][1] == 7
                    and isinstance(node[0][0], str) and isinstance(node[3], str)):
                add(node[0][0], node[3], num(g(node, 4, 1)), "page1")
                return
            for v in node:
                walk(v)
    for m in re.finditer(r"AF_initDataCallback\(\{key: '(ds:\d+)', hash: '\d+', data:(.*?), sideChannel: \{\}\}\);</script>", html, re.S):
        try:
            data = json.loads(m.group(2))
        except ValueError:
            continue
        for sec in g(data, 0, 1) or []:
            if (text(g(sec, 3, 1, 0, 0)) or "").startswith("search_results_cluster"):
                found = True
                walk(g(sec, 22, 0))
                tok = g(sec, 22, 1, 3, 1)
                token = tok if isinstance(tok, str) and tok else token
    if not found:
        raise RuntimeError(f"developer page for {name!r} has no results section; Play markup changed")
    page = 1
    while token and len(results) < max_results and page < max_pages:
        time.sleep(pause)
        page += 1
        cards, token = next_page(token)
        for c in cards:
            add(text(g(c, 12, 0)), text(g(c, 2)), num(g(c, 6, 0, 2, 1, 1)), f"page{page}")
    complete = token is None and len(results) <= max_results
    if not complete:
        print(f"developer_apps({name!r}): stopped by a cap (max_results={max_results}, max_pages={max_pages}); more apps exist")
    return results[:max_results], complete

for app_id in ["com.google.android.youtube", "com.microsoft.teams", "com.rovio.baba", "com.mojang.minecraftpe"]:
    name = developer_name(app_id)
    time.sleep(1)
    apps, complete = developer_apps(name) if name else ([], True)
    pages = {}
    for a in apps:
        pages[a["source"]] = pages.get(a["source"], 0) + 1
    print(app_id, "->", repr(name), len(apps), "complete:", complete, pages, [a["app_id"] for a in apps[:4]])
    time.sleep(1)

print("first row:", json.dumps(apps[0]))
time.sleep(1)
print("capped:", len(developer_apps("Google LLC", max_results=30)[0]))
time.sleep(1)
print("wrong name:", developer_apps("Rovio Entertainment Corporation"))
```

Output (from the run on 2026-10-23):

```text
com.google.android.youtube -> 'Google LLC' 90 complete: True {'page1': 20, 'page2': 50, 'page3': 20} ['com.android.chrome', 'com.google.android.apps.messaging', 'com.google.android.googlequicksearchbox', 'com.google.android.gm']
com.microsoft.teams -> 'Microsoft Corporation' 72 complete: True {'page1': 20, 'page2': 50, 'page3': 2} ['com.microsoft.office.outlook', 'com.microsoft.teams', 'com.microsoft.office.officehubrow', 'com.microsoft.skydrive']
com.rovio.baba -> 'Rovio Entertainment Oy' 16 complete: True {'page1': 16} ['com.rovio.baba', 'com.rovio.angrybirdsfriends', 'com.rovio.BadPiggies', 'com.rovio.dream']
com.mojang.minecraftpe -> 'Mojang' 3 complete: True {'page1': 3} ['com.mojang.minecraftpe', 'com.mojang.minecraftedu', 'com.mojang.minecraftedu_preview']
first row: {"app_id": "com.mojang.minecraftpe", "title": "Minecraft: Dream it, Build it!", "rating": 4.303237, "source": "page1"}
developer_apps('Google LLC'): stopped by a cap (max_results=30, max_pages=20); more apps exist
capped: 30
wrong name: None
```

## Fields

Detail record from `play_app()`, exactly these keys. `a` is the app array `data[1][2]` of the callback whose `a[77][0]` equals the package name.

| Field | Source | Example (observed) |
|---|---|---|
| `app_id` | the package name you asked for | `com.spotify.music` |
| `title` | `a[0][0]` (fallback JSON-LD `name`) | `Spotify: Music and Podcasts` |
| `developer` | `a[68][0]` (fallback JSON-LD `author.name`) | `Spotify AB` |
| `developer_id` | `a[68][2]` | `6523013212965523194` |
| `rating` | `a[51][0][1]` | `4.3489933` |
| `ratings_count` | `a[51][2][1]` | `36528080` |
| `installs` | `a[13][0]` | `1,000,000,000+` |
| `min_installs` / `installs_exact` | `a[13][1]` / `a[13][2]` | `1000000000` / `3171683950` |
| `price`, `currency`, `price_text` | `a[57][0][0][0][0][1][0]` = `[micros, currency, text]`; the text is empty for free apps, so `price_text` becomes `Free` when micros is 0 | `6.99`, `USD`, `$6.99` (Minecraft) |
| `category` / `category_id` | `a[79][0][0][0]` / `a[79][0][0][2]` | `Arcade` / `GAME_ARCADE` |
| `updated` / `updated_ts` | `a[145][0][0]` / `a[145][0][1][0]` (unix s) | `Oct 23, 2026` / `1792759042` |
| `version` | `a[140][0][0][0]`, null means "Varies with device" | `1.26.52.3` |
| `content_rating` | `a[9][0]` (also JSON-LD `contentRating`) | `Everyone 10+` |
| `released` | `a[10][0]` | `May 27, 2014` |
| `contains_ads` | `a[48][0]` present | `true` |
| `in_app_purchases` | `a[19][0]` | `$4.99 - $203.88 per item` |
| `url` | the detail URL that was fetched | `https://play.google.com/store/apps/details?id=com.spotify.music&hl=en_US&gl=US` |

Search records from `play_search()`:

| Card | Keys |
|---|---|
| top result (`section[23]`) | `app_id`, `title`, `source` |
| page 1 | `app_id` (`c[0][0]`), `title` (`c[3]`), `developer` (`c[14]`), `rating` (`c[4][1]`), `category` (`c[5]`), `installs` (`c[15]`), `price_text` (`c[8][1][0]`, same rule as above), `source` |
| continuation pages | `app_id` (`c[12][0]`), `title` (`c[2]`), `developer` (`c[4][0][0][0]`), `rating` (`c[6][0][2][1][1]`), `source` |

`source` says where each hit came from (`top_result`, `page1`, `page2`, ...). An unrated app has `rating: null`.

Developer list from `developer_apps()`: `app_id`, `title`, `rating` and `source`, read from the same card shapes as search (page 1 `c[0][0]`, `c[3]`, `c[4][1]`; continuation pages `c[12][0]`, `c[2]`, `c[6][0][2][1][1]`). `developer_name()` returns JSON-LD `author.name` from the app page.

## Gotchas

- **Positional arrays, not named keys.** The `AF_initDataCallback` data is nested lists. Indexes above were correct on 2026-10-23; when Play ships a new frontend they can move. The code uses a safe getter `g()` plus type checks (`num()`, `text()`) so a moved field becomes `None` instead of a crash or a wrong type; check for a sudden column of `None`s. JSON-LD fields are the stable fallback for title, developer, rating, ratings count, price, category id and content rating.
- **Callback keys are not stable labels.** The app data was in `ds:5` on the pages tested, but the code finds the callback by matching the package name at `a[77][0]` rather than trusting the key. JSON-LD is only used when its `url` carries the same `id`.
- **Brand queries put the main app in a separate "top result" card** (`section[23]`), not in the result list. Searching `spotify` returned `com.spotify.music` only there; `play_search()` adds it first with `source: "top_result"`.
- **Search pages also contain a `similar_apps` section** (e.g. Roblox under `minecraft`) and related searches. `play_search()` only takes sections named `search_results_cluster_*` plus the top result. A naive collection of every `details?id=` link (block 2's rendered list included) can mix these in.
- **Scrolling does not paginate search.** In the cloud browser, scrolling `pdf scanner` (19 ids) and `minecraft` (40 ids) to the bottom loaded nothing new; the page ends in "Related searches". The only batchexecute call the page made was `teXCtc`, not a results call. More results exist only through the `qnKhOb` token, and only when the page includes one (`section[22][1][3][1]`).
- **`qnKhOb` is an undocumented internal RPC.** It answered plain POSTs with `)]}'` followed by a `wrb.fr` line whose third element is a JSON string. Its next token sits at `payload[0][0][7][1]`; the last page has none. The token travels inside the JSON `f.req` value, which `urlencode` percent-encodes in full. Expect the RPC to change without notice: `next_page()` raises instead of returning a short list, so a broken chain is never mistaken for the end.
- **Caps are explicit.** `max_results` and `max_pages` stop the walk early; the function then prints a `stopped by a cap` line and returns `complete=False`. `minecraft` with `max_results=80` stopped that way; with `max_results=1000` it ran to the natural end at 250 and returned `complete=True`.
- **For queries capped at ~30, widen with more queries**, not more pages: keyword variants, or developer pages (block 3).
- **Developer page names must match exactly.** `Rovio Entertainment Corporation` returned HTTP 404; the name on the Angry Birds 2 page is `Rovio Entertainment Oy`. Take the name from the app page (`developer_name()`, or `developer` from `play_app()`), never from memory. `developer_apps()` returns `None` on a 404, so "no such page" is distinct from an empty list.
- **Two kinds of developer page.** App pages link either to `/store/apps/developer?id=<name>` (Spotify) or to `/store/apps/dev?id=<numeric id>` (Microsoft Teams, Angry Birds 2, YouTube). The numeric page is a showcase: 10-11 apps, no paginated results section. The name page worked for those numeric developers too (Microsoft Corporation: 72 apps), so block 3 always uses the name page.
- **Developer lists are near-complete, not proven complete.** The result section name ends in a number that looks like a total (`search_results_cluster_apps_range_0_92` for Google LLC, `_74` for Microsoft Corporation, `_19` for Rovio Entertainment Oy) a little above what came back (90, 72, 16). Which apps make up the gap was not determined. `complete=True` means the token chain ended, not that this gap is closed.
- **Country and language come from `gl` and `hl`.** With `GL = "DE"` Minecraft came back as `6.99 EUR`, `€6.99`. Set them explicitly; without them Play picks from the request.
- **Missing apps return HTTP 404** (`com.example.does.not.exist`). `fetch()` raises `urllib.error.HTTPError` at once for a 404 (only 429 and 5xx are retried); `play_app()` turns the 404 into `None`, and also returns `None` for any page with neither app data nor matching JSON-LD. Apps unavailable in the chosen country were not tested.
- **Plain urllib, not the harness `http_get`.** `http_get` routes through a proxy when `BROWSER_USE_API_KEY` is set, which can change status-code behaviour; blocks 1 and 3 call urllib directly so 404, 429 and 5xx are seen as they are. No 429 or 5xx occurred today, so the retry path was not exercised against Play; it was checked against a local test server (429 with `Retry-After: 1`, 503 with an HTTP-date `Retry-After`, a persistent 503 that raises, a 404 that raises at once).
- **Numbers are live.** Spotify's ratings count changed between fetches minutes apart. Store a fetch timestamp with every row.
- **Rendered and HTTP result counts can differ.** `weather` gave 30 package names over HTTP; the rendered page gave 25 on some loads and 30 on others. Use the HTTP path when you want the full page-1 list.
- **Missing app in the browser:** the page title becomes `Not Found` and there is no JSON-LD block; block 2 reports that as `not_found: true` with `null` fields.
- **`version` is often null** for apps that ship per-device builds (Spotify, VLC). That is the page's "Varies with device", not a parse failure.
- **The harness prefixes the tab title with a horse marker** while attached, so `document.title` is not the app name; use JSON-LD `name`.
- **Pacing:** 1 request per 0.5-1 s was never throttled today (25 detail pages at 1 s, 21 at 0.5 s, 5 RPC pages at 1 s, repeated across several runs). Keep at least that spacing.

## What does not work

- **Scroll-to-load on search** in the browser: no extra results (tested on two queries, 8-15 scrolls each).
- **More than ~30 results for most queries**: 12 of 13 queries tested had no continuation token, so page 1 is everything Play will return for them.
- **More than 250 results** even when paginating: the `minecraft` token chain ended after 250 apps.
- **Ratings count, last updated and version from search cards**: not present; fetch the detail page.
- **Full catalogs from numeric `/store/apps/dev?id=` pages**: 10-11 apps and no paginated results section for the three tested (Microsoft, Rovio, Google). Use the name page.
- **Guessed developer names**: HTTP 404 unless the name matches Play's spelling exactly.
- **User reviews**: not covered; no reviews endpoint was tested.
- **Login-only data** (your library, purchases, personalised recommendations) was not attempted; this skill covers public logged-out pages only.
