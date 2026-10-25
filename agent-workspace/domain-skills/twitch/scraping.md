# Twitch — Live Channels by Category & Channel About Info

Field-tested against www.twitch.tv on 2026-10-24 using a Browser Use cloud browser (directory and channel pages) and plain urllib (channel HTML).

robots.txt (fetched 2026-10-24) for `User-agent: *` allows `/directory`, `/<login>` and `/<login>/about` and disallows `/search`, `/embed/`, `/popout/`, `/chat/` and account pages; `gql.twitch.tv/robots.txt` and `api.twitch.tv/robots.txt` are both `Disallow: /`, so this skill only loads www.twitch.tv pages and never calls those hosts itself.

## Quick summary

- **Live channels for a category** (`/directory/category/<slug>?sort=VIEWER_COUNT`): the HTML is a 222 KB app shell with no stream data, so block 1 renders the page in the browser and reads the channel cards (login, display name, title, viewer count, tags, verified, thumbnail). Infinite scroll adds ~24 cards per step and never removes earlier ones: 150 (VALORANT) and 300 (Just Chatting) channels collected in one tab, and a small category (Super Mario Maker 2) ran to its end at 17 channels (3 scroll steps with no new cards).
- **Category slugs** (`/directory`): block 2 reads the box-art cards (name, slug, viewers, tags). 100 cards collected; 2 of them were `/directory/event/...` pages, not categories.
- **Channel about info, no browser** (`/<login>`): plain HTTP GET returned 200 with a JSON-LD `ProfilePage` holding display name, bio, profile image, **exact follower count**, social links and recent VODs/clips; `og:title`/`og:description` add live status, stream title and category (no usable viewer count, see Gotchas). 23 channel fetches at ~1 request/s: 23 x HTTP 200, no 429.
- **Channel about info, browser** (`/<login>/about` + `/<login>`): panels (images, text, links), Partner badge, and the live header with the **exact** current viewer count, uptime and the full tag list.

## What works

### 1. Live channels in a category, many pages (browser)

```python
import json, re, time, urllib.parse

CATEGORY_SLUG = "valorant"      # the <slug> in /directory/category/<slug> (block 2 lists them)
SORT = "VIEWER_COUNT"           # RELEVANCE | VIEWER_COUNT | VIEWER_COUNT_ASC | RECENT
MAX_CHANNELS = 150              # ~24-26 cards load per scroll step
PAUSE = 2.0                     # seconds between scroll steps (polite pacing)

BASE = "https://www.twitch.tv/directory/category/" + urllib.parse.quote(CATEGORY_SLUG, safe="")
URL = BASE + "?" + urllib.parse.urlencode({"sort": SORT})

def open_page(url, timeout=20):
    """Navigate the attached tab (new tab only if it is not on twitch yet); confirm the path before reading."""
    want = url.split("?")[0]
    try:
        href = js("location.href") or ""
    except Exception:
        href = ""
    if href.startswith("https://www.twitch.tv/"):
        goto_url(url)
    else:
        new_tab(url)
    deadline = time.time() + timeout
    while True:
        href = js("location.href") or ""
        if href.split("?")[0].split("#")[0] == want and js("document.readyState") == "complete":
            return href
        if time.time() > deadline:
            raise RuntimeError(f"expected {want}, tab is on {href}")
        time.sleep(0.5)

CARDS_JS = r"""(() => [...document.querySelectorAll('article')].map(art => {
  const link = art.querySelector('a[data-a-target="preview-card-channel-link"]');
  const img = art.querySelector('a[data-a-target="preview-card-image-link"]');
  if (!link || !img) return null;                     // skips the "May Contain Labeled Content" notice
  const title = link.querySelector('h4');
  const name = link.querySelector('p[title]');
  const stat = img.querySelector('.tw-media-card-stat');
  return {
    login: (link.getAttribute('href') || '').replace(/^\//, ''),
    display_name: name ? name.getAttribute('title') : null,
    title: title ? (title.getAttribute('title') || title.textContent) : null,
    viewers_text: stat ? stat.textContent.trim() : null,
    tags: [...art.querySelectorAll('button.tw-tag')].map(b => b.textContent.trim()),
    verified: [...link.querySelectorAll('svg title')].some(t => t.textContent === 'Verified'),
    thumbnail: (img.querySelector('img.tw-image') || {}).src || null,
  };
}).filter(Boolean))()"""

STATE_JS = r"""(() => {
  const t = (document.querySelector('main') || document.body).innerText;
  if (/Category does not exist/.test(t)) return 'missing';
  if (/No results available/.test(t)) return 'empty'; // category exists, nobody live right now
  return '';
})()"""

def viewers_to_int(s):
    """'27.3K viewers' -> 27300, '762 viewers' -> 762, '1 viewer' -> 1 (Twitch rounds K/M)."""
    m = re.match(r"([\d.,]+)\s*([KM]?)", s or "")
    if not m:
        return None
    n = float(m.group(1).replace(",", ""))
    return int(round(n * {"": 1, "K": 1_000, "M": 1_000_000}[m.group(2)]))

open_page(URL)

# Wait for the first cards (the grid renders client-side after load).
deadline, empty_polls = time.time() + 20, 0
while not js(CARDS_JS):
    state = js(STATE_JS)
    if state == "missing":
        raise SystemExit(f"Twitch says category '{CATEGORY_SLUG}' does not exist")
    empty_polls = empty_polls + 1 if state == "empty" else 0
    if empty_polls >= 3:      # stable empty state, not a loading flash
        break
    if time.time() > deadline:
        raise RuntimeError("no channel cards and no empty-state message within 20 s")
    time.sleep(1)

# Infinite scroll: wheel over the grid until MAX_CHANNELS or the list stops growing.
seen, stalls = {}, 0
while empty_polls < 3 and len(seen) < MAX_CHANNELS and stalls < 3:
    before = len(seen)
    for c in js(CARDS_JS):
        seen.setdefault(c["login"], c)
    if len(seen) == before:
        stalls += 1
    else:
        stalls = 0
    if len(seen) >= MAX_CHANNELS:
        break
    scroll(js("Math.round(innerWidth * 0.55)"), 450, dy=3000)   # pointer over the grid, not the sidebar
    time.sleep(PAUSE)

channels = list(seen.values())[:MAX_CHANNELS]
for pos, c in enumerate(channels, 1):
    c["grid_position"] = pos          # order the cards were appended while scrolling
    c["viewers"] = viewers_to_int(c["viewers_text"])
    c["url"] = "https://www.twitch.tv/" + c["login"]
# Counts change while you scroll, so grid order is only roughly sorted; re-sort by the card counts.
if SORT in ("VIEWER_COUNT", "VIEWER_COUNT_ASC"):
    channels.sort(key=lambda c: c["viewers"] if c["viewers"] is not None else -1, reverse=SORT == "VIEWER_COUNT")
for rank, c in enumerate(channels, 1):
    c["rank"] = rank

print(len(channels), "live channels in", CATEGORY_SLUG, "sorted by", SORT)
print(json.dumps(channels[:2], indent=1, ensure_ascii=False))
```

Real output (trimmed; channel logins and stream titles replaced):

```
150 live channels in valorant sorted by VIEWER_COUNT
[
 {
  "login": "<login>",
  "display_name": "<display name>",
  "title": "<stream title>",
  "viewers_text": "3.6K viewers",
  "tags": ["English", "<tag>", "<tag>"],
  "verified": true,
  "thumbnail": "https://static-cdn.jtvnw.net/previews-ttv/live_user_<login>-640x360.jpg",
  "grid_position": 1,
  "viewers": 3600,
  "url": "https://www.twitch.tv/<login>",
  "rank": 1
 },
 ...
]
```

Sort values observed in the URL and the resulting order (VALORANT): `RELEVANCE` (sort control reads "Recommended For You"; mixed: 3.1K, 762, 2.8K, 61 ...), `VIEWER_COUNT` ("Viewers (High to Low)"; 3K, 2.8K, 2.6K ...), `VIEWER_COUNT_ASC` ("Viewers (Low to High)"; 0, 0, 0, 1 ...), `RECENT` (newest streams, 0-1 viewers). With no `sort` param the default was not stable: one load showed "Recommended For You", another load the same day showed "Viewers (High to Low)". Always pass `sort`.

With `VIEWER_COUNT`, the raw grid order is not strictly descending over 150+ cards (counts move during the 1-2 minutes of scrolling), so the block keeps that order as `grid_position` and re-sorts by the card count for `rank`.

### 2. List categories to get slugs (browser)

```python
import json, re, time

MAX_CATEGORIES = 100   # ~30 box-art cards load per scroll step
PAUSE = 2.0

def open_page(url, timeout=20):
    """Navigate the attached tab (new tab only if it is not on twitch yet); confirm the path before reading."""
    want = url.split("?")[0]
    try:
        href = js("location.href") or ""
    except Exception:
        href = ""
    if href.startswith("https://www.twitch.tv/"):
        goto_url(url)
    else:
        new_tab(url)
    deadline = time.time() + timeout
    while True:
        href = js("location.href") or ""
        if href.split("?")[0].split("#")[0] == want and js("document.readyState") == "complete":
            return href
        if time.time() > deadline:
            raise RuntimeError(f"expected {want}, tab is on {href}")
        time.sleep(0.5)

CATS_JS = r"""(() => [...document.querySelectorAll('div.game-card')].map(c => {
  const a = c.querySelector('a[data-a-target="tw-box-art-card-link"]');
  const h = c.querySelector('[data-a-target="tw-card-title"] h2');
  const v = c.querySelector('p a[aria-label*="viewers for"]');
  if (!a || !h) return null;
  return {
    name: h.getAttribute('title') || h.textContent,
    path: a.getAttribute('href'),
    viewers_text: v ? v.textContent.trim() : null,
    tags: [...c.querySelectorAll('.tw-tag')].map(t => t.textContent.trim()),
    box_art: (a.querySelector('img') || {}).src || null,
  };
}).filter(Boolean))()"""

def viewers_to_int(s):
    m = re.match(r"([\d.,]+)\s*([KM]?)", s or "")
    if not m:
        return None
    return int(round(float(m.group(1).replace(",", "")) * {"": 1, "K": 1_000, "M": 1_000_000}[m.group(2)]))

open_page("https://www.twitch.tv/directory")
deadline = time.time() + 20
while not js(CATS_JS):
    if time.time() > deadline:
        raise RuntimeError("no category cards rendered within 20 s")
    time.sleep(1)

seen, stalls = {}, 0
while len(seen) < MAX_CATEGORIES and stalls < 3:
    before = len(seen)
    for c in js(CATS_JS):
        seen.setdefault(c["path"], c)
    stalls = stalls + 1 if len(seen) == before else 0
    if len(seen) >= MAX_CATEGORIES:
        break
    scroll(js("Math.round(innerWidth * 0.55)"), 450, dy=3000)   # pointer over the grid, not the sidebar
    time.sleep(PAUSE)

cats = []
for rank, c in enumerate(list(seen.values())[:MAX_CATEGORIES], 1):
    kind, _, slug = c["path"].strip("/").partition("/")[2].partition("/")   # "category"/"event", slug
    cats.append({**c, "rank": rank, "kind": kind, "slug": slug,
                 "viewers": viewers_to_int(c["viewers_text"]),
                 "url": "https://www.twitch.tv" + c["path"]})

print(len(cats), "categories;", sum(c["kind"] == "category" for c in cats), "are /directory/category/ pages")
print(json.dumps(cats[:3], indent=1, ensure_ascii=False))
```

Real output (trimmed):

```
100 categories; 98 are /directory/category/ pages
[
 {"name": "Stream-o-Ween", "path": "/directory/event/stream-o-ween-2026", "viewers_text": "35.8K viewers", "tags": [], "kind": "event", "slug": "stream-o-ween-2026", "viewers": 35800, ...},
 {"name": "Shopping", "path": "/directory/event/shopping-2026", "viewers_text": "4K viewers", "kind": "event", ...},
 {"name": "Just Chatting", "path": "/directory/category/just-chatting", "viewers_text": "243K viewers", "tags": ["IRL"],
  "box_art": "https://static-cdn.jtvnw.net/ttv-boxart/509658-285x380.jpg", "rank": 3, "kind": "category", "slug": "just-chatting", "viewers": 243000,
  "url": "https://www.twitch.tv/directory/category/just-chatting"}
]
```

`/directory` is ordered by Twitch's recommendation, not strictly by viewers (rank 98-100 had 375, 3.1K, 3.2K viewers).

### 3. Channel about info from the HTML (no browser, preferred)

```python
import gzip, html, json, re, time, urllib.error, urllib.parse, urllib.request

LOGINS = ["twitchgaming", "twitch", "zzqx_no_such_channel_9871"]
UA = "twitch-channel-about/1.0 (python-urllib; one request per second)"   # truthful client UA
PAUSE = 1.0

def fetch(url, max_wait=120):
    """GET with capped exponential backoff on 429/5xx (total wait <= max_wait s, no sleep after the last try)."""
    waited, delay = 0, 2
    while True:
        req = urllib.request.Request(url, headers={"User-Agent": UA, "Accept-Encoding": "gzip", "Accept-Language": "en-US"})
        try:
            with urllib.request.urlopen(req, timeout=20) as r:
                data = r.read()
                if r.headers.get("Content-Encoding") == "gzip":
                    data = gzip.decompress(data)
                return data.decode("utf-8")
        except urllib.error.HTTPError as e:
            if e.code not in (429, 500, 502, 503, 504) or waited + delay > max_wait:
                raise
        time.sleep(delay)
        waited += delay
        delay *= 2

def meta(page, key):
    for pat in (rf'<meta (?:property|name)="{key}" content="([^"]*)"', rf'<meta content="([^"]*)" (?:property|name)="{key}"'):
        m = re.search(pat, page)
        if m:
            return html.unescape(m.group(1))
    return None

def channel_about(login, attempts=4, budget=120):
    url = "https://www.twitch.tv/" + urllib.parse.quote(login.lower(), safe="")
    deadline = time.time() + budget          # all retries for one login share a 120 s budget
    for attempt in range(attempts):
        page = fetch(url, max_wait=max(0, deadline - time.time()))
        graph = []
        for block in re.findall(r'<script type="application/ld\+json">(.*?)</script>', page, re.S):
            graph += json.loads(block).get("@graph", [])
        profile = next((g for g in graph if g.get("@type") == "ProfilePage"), None)
        if profile:
            break
        # Unknown (or unavailable) login: Twitch's generic page, og:title "Twitch" and robots "noindex".
        # A real channel never carries both, so this is a positive "missing" signal, not a fallback.
        if meta(page, "og:title") == "Twitch" and meta(page, "robots") == "noindex":
            return {"login": login, "error": "no such channel (or unavailable)"}
        # Anything else (e.g. a ~217 KB shell with no meta tags and no JSON-LD) is transient: refetch.
        wait = 5 * 2 ** attempt              # 5, 10, 20 s
        if attempt == attempts - 1 or time.time() + wait > deadline:
            break
        time.sleep(wait)
    if not profile:
        raise RuntimeError(f"{url}: no ProfilePage JSON-LD and no 'missing channel' marker after {attempt + 1} tries")
    person = profile.get("mainEntity") or {}
    stat = person.get("interactionStatistic") or {}
    videos = next((g for g in graph if g.get("@type") == "ItemList"), {}).get("itemListElement", [])
    title = meta(page, "og:title") or ""
    desc = meta(page, "og:description") or ""
    live = title.endswith(" - Live on Twitch")
    # Live og:description is "<title> | Streaming <category> for N viewers." OR "<title> | Streaming <category>."
    # N is NOT used: it was off by up to 300x versus the live count (see Gotchas). Use block 1 or 4 for viewers.
    m = re.match(r"(.*)\|\s*Streaming (.+?)(?: for \d+ viewers?)?\.$", desc, re.S) if live else None
    if live and not m:
        raise RuntimeError(f"{url}: live, but og:description has an unknown shape: {desc[-80:]!r}")
    return {
        "login": person.get("alternateName") or login,
        "display_name": person.get("name"),
        "url": person.get("url") or url,
        "bio": person.get("description"),
        "profile_image": person.get("image"),
        "followers": stat.get("userInteractionCount"),
        "social_links": person.get("sameAs", []),
        "live": live,
        "live_title": m.group(1).strip() if m else None,
        "live_category": m.group(2) if m else None,   # lowercase category name, e.g. "just chatting"
        "recent_videos": [{"title": v.get("name"), "url": v.get("url"), "uploaded": v.get("uploadDate"),
                           "duration": v.get("duration"),
                           "views": (v.get("interactionStatistic") or {}).get("userInteractionCount")}
                          for v in videos if v.get("@type") == "VideoObject"],
        "page_modified": profile.get("dateModified"),
    }

out = []
for i, login in enumerate(LOGINS):
    if i:
        time.sleep(PAUSE)
    out.append(channel_about(login))
print(json.dumps([{**o, "recent_videos": o["recent_videos"][:2]} if "recent_videos" in o else o for o in out], indent=1, ensure_ascii=False))
```

Real output (trimmed):

```
[
 {
  "login": "twitchgaming",
  "display_name": "twitchgaming",
  "url": "https://www.twitch.tv/twitchgaming",
  "bio": "Join /twitchgaming and be a part of the conversation! ...",
  "profile_image": "https://static-cdn.jtvnw.net/jtv_user_pictures/77eb821f-...-profile_image-300x300.jpeg",
  "followers": 584895,
  "social_links": ["https://twitter.com/twitch", "https://instagram.com/twitch"],
  "live": false, "live_title": null, "live_category": null,
  "recent_videos": [],
  "page_modified": "2023-10-15T02:56:52Z"
 },
 {
  "login": "twitch", "display_name": "Twitch", "followers": 2483231,
  "social_links": ["https://twitter.com/twitch", "https://www.instagram.com/twitch/", "https://www.tiktok.com/@twitch", "https://www.youtube.com/twitch"],
  "recent_videos": [
   {"title": "Let's Chat Episode 14 | ...", "url": "https://www.twitch.tv/videos/2908640776", "uploaded": "2026-10-23T16:55:16Z", "duration": "PT3353S", "views": 8},
   ...
  ], ...
 },
 {"login": "zzqx_no_such_channel_9871", "error": "no such channel (or unavailable)"}
]
```

For live channels the same code returned `"live": true`, `"live_title": "<stream title>"` and `"live_category"` as the lowercase category name (`"valorant"`, `"just chatting"`). `og:description` came in two shapes, and the regex handles both: `"<title> | Streaming valorant for 234 viewers."` and `"<title> | Streaming valorant."` (4 of 13 live VALORANT channels had no count). The HTML never gives a usable viewer count; take viewers from block 1 (card) or block 4 (exact).

### 4. Channel about panels and exact live stats (browser)

```python
import json, re, time, urllib.parse

LOGINS = ["twitchgaming", "twitch"]   # channel logins, e.g. the "login" field from block 1
PAUSE = 2.0                           # seconds between page loads

def open_page(url, timeout=20):
    """Navigate the attached tab (new tab only if it is not on twitch yet); confirm the path before reading."""
    want = url.split("?")[0]
    try:
        href = js("location.href") or ""
    except Exception:
        href = ""
    if href.startswith("https://www.twitch.tv/"):
        goto_url(url)
    else:
        new_tab(url)
    deadline = time.time() + timeout
    while True:
        href = js("location.href") or ""
        if href.split("?")[0].split("#")[0] == want and js("document.readyState") == "complete":
            return href
        if time.time() > deadline:
            raise RuntimeError(f"expected {want}, tab is on {href}")
        time.sleep(0.5)

ABOUT_JS = r"""(() => {
  const main = document.querySelector('main') || document.body;
  if (/Sorry\. Unless you've got a time machine, that content is unavailable/.test(main.innerText)) return {missing: true};
  const p = document.querySelector('[data-a-target="about-panel"]');
  if (!p || !p.querySelector('h3')) return null;   // panel shell renders before its content
  const f = [...p.querySelectorAll('div')].find(d => / followers?$/.test(d.textContent.trim()) && d.children.length <= 1);
  return {
    display_name: (document.querySelector('main h1') || {}).textContent || null,
    about_heading: (p.querySelector('h3') || {}).textContent || null,
    followers_text: f ? f.textContent.trim() : null,
    bio: (p.querySelector('p[dir="auto"]') || {}).textContent || null,
    partner: !!p.querySelector('svg[aria-label="Verified Partner"]'),
    social_links: [...p.querySelectorAll('a[role="link"][target="_blank"]')].map(a => ({label: a.textContent.trim(), url: a.href})),
    panels: [...document.querySelectorAll('[data-test-selector="channel_panel_test_selector"]')].map(x => ({
      text: x.innerText.trim().slice(0, 500),
      images: [...x.querySelectorAll('img')].map(i => ({src: i.src, alt: i.alt || null})),
      links: [...x.querySelectorAll('a[href^="http"]')].map(a => a.href),
    })),
  };
})()"""

LIVE_JS = r"""(() => {
  const t = s => { const e = document.querySelector(s); return e ? e.textContent.trim() : null; };
  const title = t('[data-a-target="stream-title"]');
  if (!title) return {live: false};
  const g = document.querySelector('[data-a-target="stream-game-link"]');
  return {
    live: true,
    stream_title: title,
    category: g ? g.textContent.trim() : null,
    category_url: g ? g.href : null,
    viewers_text: t('[data-a-target="animated-channel-viewers-count"]'),
    uptime: ((document.querySelector('.live-time') || {}).innerText || '').split('\n')[0] || null,
    tags: [...document.querySelectorAll('main .tw-tag')].map(b => b.textContent.trim()),
  };
})()"""

def count(s):
    """'585K followers' -> 585000, '1,175' -> 1175 (follower counts are rounded by Twitch)."""
    m = re.match(r"([\d.,]+)\s*([KM]?)", s or "")
    if not m:
        return None
    return int(round(float(m.group(1).replace(",", "")) * {"": 1, "K": 1_000, "M": 1_000_000}[m.group(2)]))

def poll(expr, timeout=20):
    deadline = time.time() + timeout
    while time.time() < deadline:
        r = js(expr)
        if r:
            return r
        time.sleep(1)
    return None

results = []
for i, login in enumerate(LOGINS):
    if i:
        time.sleep(PAUSE)
    base = "https://www.twitch.tv/" + urllib.parse.quote(login.lower(), safe="")
    open_page(base + "/about")
    about = poll(ABOUT_JS)
    if about is None:
        raise RuntimeError(f"about panel for {login} did not render within 20 s")
    if about.get("missing"):   # unknown logins: "Sorry. Unless you've got a time machine, that content is unavailable."
        results.append({"login": login, "error": "content unavailable"})
        continue
    about["followers"] = count(about["followers_text"])
    time.sleep(PAUSE)
    open_page(base)
    time.sleep(4)   # the live header renders a few seconds after load; offline channels never show it
    live = js(LIVE_JS)
    if live.get("live"):
        live["viewers"] = count(live["viewers_text"])
    results.append({"login": login, "url": base, **about, **live})

print(json.dumps([{**r, "panels": r.get("panels", [])[:2]} for r in results], indent=1, ensure_ascii=False))
```

Real output (trimmed):

```
[
 {
  "login": "twitchgaming",
  "url": "https://www.twitch.tv/twitchgaming",
  "display_name": "twitchgaming",
  "about_heading": "About twitchgaming",
  "followers_text": "585K followers",
  "bio": "Join /twitchgaming and be a part of the conversation! ...",
  "partner": true,
  "social_links": [{"label": "Twitter", "url": "https://twitter.com/twitch"}, {"label": "Instagram", "url": "https://instagram.com/twitch"}],
  "panels": [{"text": "", "images": [...], "links": []}, ...],
  "followers": 585000,
  "live": false
 },
 {"login": "twitch", "display_name": "Twitch", "followers_text": "2.5M followers", "partner": true,
  "panels": [{"text": "", "images": [{"src": "https://panels.twitch.tv/panel-12826-image-...", "alt": "Panel Content"}], "links": ["https://twitter.com/Twitch"]}, ...],
  "live": false, ...}
]
```

On a live channel the live part read: `"live": true, "stream_title": "<stream title>", "category": "VALORANT", "category_url": "https://www.twitch.tv/directory/category/valorant", "viewers_text": "2,546", "viewers": 2546, "uptime": "7:03:55", "tags": [10 tags]`. Two more live channels gave 3,586 and 359 viewers with 4 and 1 tags.

## Fields

| Field | Source | Notes |
|---|---|---|
| `login` | block 1 card `a[data-a-target="preview-card-channel-link"]` href; block 3 JSON-LD `alternateName` | lowercase, unique key; URL is `https://www.twitch.tv/<login>` |
| `display_name` | card `p[title]`; JSON-LD `mainEntity.name`; about `main h1` | may differ in case or script from `login` |
| `title` / `stream_title` / `live_title` | card `h4[title]`; `[data-a-target="stream-title"]`; `og:description` before ` \| Streaming ` | full text in the `title` attribute |
| `viewers_text`, `viewers` | card `.tw-media-card-stat` (`"27.3K viewers"`, rounded); channel page `[data-a-target="animated-channel-viewers-count"]` (`"1,175"`, exact) | block 1 converts K/M, so card counts are approximate; block 3 has no viewer field on purpose |
| `live_category` (block 3) | `og:description` after ` \| Streaming ` | lowercase category name, not the URL slug (`"just chatting"`) |
| `rank`, `grid_position` (block 1) | position after re-sorting by `viewers`; raw card append order | they differ for `VIEWER_COUNT` on long lists |
| `tags` | card `button.tw-tag`; channel page `main .tw-tag` | cards showed 1-5 tags, the channel page showed all (10 for one channel whose card showed 3) |
| `verified` | card `svg > title` = `Verified` | |
| `partner` | about panel `svg[aria-label="Verified Partner"]` | |
| `thumbnail` | card `img.tw-image` | `previews-ttv/live_user_<login>-640x360.jpg` |
| `category`, `category_url` | `[data-a-target="stream-game-link"]` | display name and `/directory/category/<slug>` |
| `uptime` | `.live-time` first line | `H:MM:SS` |
| `followers` | JSON-LD `interactionStatistic.userInteractionCount` (exact, `584895`); about panel text (rounded, `585K`) | |
| `bio` | JSON-LD `mainEntity.description`; about `p[dir="auto"]` | |
| `profile_image` | JSON-LD `mainEntity.image` / `og:image` | 300x300 |
| `social_links` | JSON-LD `sameAs`; about `a[role="link"][target="_blank"]` (with labels) | |
| `panels` | `[data-test-selector="channel_panel_test_selector"]` | many are image-only (`alt: "Panel Content"`); links can be sponsor links |
| `recent_videos` | JSON-LD `ItemList` of `VideoObject` | VODs and clips: title, url, uploadDate, ISO duration, views |
| `name`, `slug`, `viewers`, `kind` (block 2) | `div.game-card` h2 `title`, href, `p a[aria-label*="viewers for"]` | `kind` is `category` or `event` |

## Gotchas

- **The grid is client-rendered.** `document.readyState == "complete"` arrives before any card exists; block 1 polls for cards (they appeared within a few seconds). The first `<article>` is a "May Contain Labeled Content" notice with no channel link; block 1 skips it.
- **Empty and missing categories.** A category with nobody live (Tetris on 2026-10-24) shows `No results available`. An unknown slug shows `Category does not exist`.
- **Scrolling** uses mouse-wheel events over the grid (`scroll(x, y, dy=3000)`); the page scrolls an inner container, not `window`. Pointing at the left sidebar scrolls the sidebar instead. 3 steps with no new cards = end of list.
- **Viewer counts on cards are rounded** (`3.2K`, `221K`). For exact numbers open the channel page (block 4).
- **The HTML viewer count is wrong, not just late.** It looks frozen from early in the stream: for 9 live VALORANT channels the `og:description` count vs the card count was e.g. 234 vs 2.9K, 677 vs 1.3K, 1 vs 365, 83 vs 364. Against block 4's player header: 234 vs 3,586, 1 vs 359, and 2,323 vs 2,546 for a channel whose HTML had said 2,323 hours earlier too (uptime 7:03:55). Block 3 therefore returns no viewer number; block 4's player header is exact and current.
- **`/about` does not show the live header.** Live title, category, uptime and exact viewers only render on `/<login>`, a few seconds after load, which is why block 4 loads both pages.
- **The about panel renders an empty shell first**; block 4 waits until its `h3` exists. Unknown logins show "Sorry. Unless you've got a time machine, that content is unavailable."
- **Transient shell pages:** 2 of 15 live channels came back HTTP 200 as a ~217 KB page with no meta tags and no JSON-LD; 4 refetches of each a minute later were all normal. Block 3 refetches (5, 10, 20 s) and raises if the profile never appears. It only reports "no such channel" on a positive marker: unknown logins return HTTP 200 with `og:title` = `Twitch` and `<meta name="robots" content="noindex">`, while real channels have `robots` = `index` and the shell page has no meta at all.
- **Consent:** the cloud browser showed a "Cookies & Ads Choices" button; it did not block cards or about pages, so the blocks leave it alone.
- **Pacing:** blocks use 2 s between scroll steps and page loads and 1 request/s for plain HTTP. No 429 or challenge was seen at that rate; block 3 backs off on 429/5xx (capped at 120 s) and then raises. If Twitch starts returning 429s or a challenge, stop.

## What does not work

- **Plain HTTP for live channel lists:** `/directory/category/<slug>` returned 200 with only `BreadcrumbList` and `CollectionPage` JSON-LD and no stream cards. Use the browser (block 1).
- **`gql.twitch.tv`** (the endpoint the page itself uses for the grid) is `Disallow: /` in its robots.txt; this skill does not call it.
- **Helix API without credentials:** `GET https://api.twitch.tv/helix/streams?first=5` returned `401 {"error":"Unauthorized","message":"OAuth token is missing"}`. The official API needs a registered app and token; not covered here.
- **Search:** `/search` is disallowed by robots.txt; find categories through `/directory` (block 2) instead.
