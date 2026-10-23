# Bing — Web Search Results Scraping

Field-tested against `www.bing.com` on 2026-10-23 using a Browser Use cloud browser and plain HTTP (`urllib`, `curl`).

## Quick summary

- **Read this first:** `https://www.bing.com/robots.txt` has `User-agent: *` / `Disallow: /search`. This skill is for person-sized lookups (a few queries, a few pages each), paced like a human. It is not a bulk SERP pipeline.
- **The official API is gone.** Microsoft's lifecycle page says "Bing Search APIs will be retired on August 11, 2025 ... decommissioned completely", and points customers to "Grounding with Bing Search as part of Azure AI Agents" (an LLM grounding tool, not a results API). `GET https://api.bing.microsoft.com/v7.0/search?q=test` without a key returned `401 PermissionDenied` in this run.
- **Use the browser, not plain HTTP.** Cookie-less `GET /search` returns 200 with 10 `li.b_algo` results, but (a) pagination is ignored, every `first=` page came back as page 1, and (b) some queries got unrelated results (`web scraping python` returned WhatsApp Web, World Wide Web - Wikipedia, Google; `scrapy tutorial` returned Microsoft account pages). The `format=rss` feed had the same problem.
- **There is no JSON.** No JSON-LD or embedded results blob. Organic results are `#b_results > li.b_algo` in server-rendered HTML; you read them from the DOM.
- **Paginate by following Bing's own Next link** (`a.sb_pagN`), never by building `first=N` yourself. In a fresh, cookie-less session a hand-made `first=11` URL was redirected (`&rdr=1&rdrig=...`) back to page 1. The Next link carries an `FPIG` token and Bing's own offsets (`first=11, 25, 39...`, `7, 21, 35...` or `4, ...`).
- **Page sizes vary: 2 to 14 organic results per page.** Page 1 of `electric cars 2026` had only 2 or 3 `li.b_algo` (answer blocks and ads took the rest), stable after load, with Next at `first=3` or `first=4`. Two 6-page runs of `web scraping python` gave 10, 6, 14, 13, 14, 14 = 71 and 10, 14, 14, 13, 14, 14 = 76 unique URLs with no CAPTCHA, paced 3-6 s per page. Dedupe on the decoded URL (a 3-page run of `electric cars 2026` had 1 repeat). The end of results is "no Next link", never "a short page".
- **Every result link is a `bing.com/ck/a?...&u=a1<base64url>` redirect.** Decode `u` to get the real URL (code below).

## What works: organic results across pages (browser)

Runs as one `browser-harness` script. Change `QUERY` and `MAX_PAGES`.

- `open_search()` marks the old document, navigates, and only returns once a *new* document is a `www.bing.com/search` page for the same `q` (not an `rdr=1` bounce) with at least one `li.b_algo`. It retries 3 times with backoff (5 s, 10 s), then raises `BingPageError`. A failed page never looks like the last page.
- Browser-call timeouts are retried: `goto_url` raises `TimeoutError`, and `js()` raises `RuntimeError("Runtime.evaluate timed out ...")` (seen once in this run right after a page load). `js_retry()` retries only those.
- `MAX_PAGES` is a safety cap. If the run stops there while Bing still shows a Next link, it prints that and `complete` stays `False`.

```python
import base64, binascii, json, random, time, urllib.parse

QUERY = "electric cars 2026"
MAX_PAGES = 3  # safety cap; the run says so when it stops here with more pages left


class BingPageError(RuntimeError):
    pass


def real_url(href):
    """Bing wraps result links as /ck/a?...&u=a1<base64url>. Return the destination (or None if there is no link)."""
    if not isinstance(href, str) or not href:
        return None
    parts = urllib.parse.urlparse(href)
    u = (urllib.parse.parse_qs(parts.query).get("u") or [""])[0]
    if parts.path == "/ck/a" and u.startswith("a1"):
        s = u[2:]
        try:
            href = base64.urlsafe_b64decode(s + "=" * (-len(s) % 4)).decode("utf-8")
        except (binascii.Error, ValueError):
            return href  # keep Bing's own redirect link (it still resolves) instead of dropping the row
    # Bing appends its own msockid tracking param to some destinations; drop it
    p = urllib.parse.urlparse(href)
    if "msockid=" in p.query:
        q = [(k, v) for k, v in urllib.parse.parse_qsl(p.query, keep_blank_values=True) if k != "msockid"]
        href = urllib.parse.urlunparse(p._replace(query=urllib.parse.urlencode(q)))
    return href


STATE = """JSON.stringify({url: location.href, stale: !!window.__bhStale, ready: document.readyState,
  n: document.querySelectorAll('#b_results > li.b_algo').length})"""


def open_search(url, query, tries=3, timeout=20):
    """Navigate to a Bing results URL and return only when the NEW document is that query's results page."""
    last = None
    for attempt in range(1, tries + 1):
        try:
            js("window.__bhStale = true")  # marks the current document; a freshly loaded one lacks it
        except (TimeoutError, RuntimeError) as e:
            last = f"could not mark the current page: {e}"  # never navigate unmarked: we could read the old page
            time.sleep(5 * attempt)
            continue
        try:
            goto_url(url)
        except TimeoutError as e:
            last = f"goto_url timed out: {e}"
        else:
            end = time.time() + timeout
            while time.time() < end:
                time.sleep(0.5)
                try:
                    st = json.loads(js(STATE))
                except (TimeoutError, RuntimeError, TypeError, ValueError) as e:  # page swapping mid-read
                    last = f"state read failed: {e}"
                    continue
                last = st
                if st["stale"] or st["ready"] != "complete":
                    continue
                p = urllib.parse.urlparse(st["url"])
                qs = urllib.parse.parse_qs(p.query)
                if p.hostname != "www.bing.com" or p.path != "/search":
                    break  # challenge, consent or redirect page: not results
                if (qs.get("q") or [""])[0].lower().split() != query.lower().split() or "rdr" in qs:
                    break  # wrong query, or Bing bounced the URL back to page 1
                if st["n"]:
                    return st
        time.sleep(5 * attempt)
    raise BingPageError(f"no Bing results page for {url} after {tries} tries; last state: {last}")


def js_retry(expr, tries=3):
    """js() timed out once right after a page load in testing (RuntimeError 'Runtime.evaluate timed out')."""
    for attempt in range(1, tries + 1):
        try:
            return js(expr)
        except (TimeoutError, RuntimeError) as e:
            if attempt == tries or "timed out" not in str(e):
                raise
            time.sleep(2 * attempt)


EXTRACT = """(() => {
  // textContent, not innerText: Bing skips rendering off-screen results, so innerText is "" for them
  const txt = el => el ? el.textContent.replace(/\\s+/g, ' ').trim() : null;
  const rows = [...document.querySelectorAll('#b_results > li.b_algo')].map(li => {
    const a = li.querySelector('h2 a');
    return {title: txt(a), href: a ? a.href : null, display_url: txt(li.querySelector('cite')),
            snippet: txt(li.querySelector('.b_caption p') || li.querySelector('p'))};
  });
  const nxt = document.querySelector('a.sb_pagN');
  return {rows, next: nxt ? nxt.getAttribute('href') : null};
})()"""

new_tab("about:blank")  # open_search does the navigation and the checks
url = "https://www.bing.com/search?" + urllib.parse.urlencode({"q": QUERY})
results, seen, complete = [], set(), False
for page in range(1, MAX_PAGES + 1):
    open_search(url, QUERY)
    data = js_retry(EXTRACT)
    if not isinstance(data, dict) or not isinstance(data.get("rows"), list):
        raise BingPageError(f"page {page}: extractor returned {data!r}")
    added = 0
    for r in data["rows"]:
        link = real_url(r.get("href"))
        if link is not None:  # rows without a link are kept, they just cannot be deduped
            if link in seen:
                continue
            seen.add(link)
        added += 1
        snippet = r.get("snippet") if isinstance(r.get("snippet"), str) else None
        if snippet and snippet.endswith("Read more"):  # some captions carry an inline "Read more" link
            snippet = snippet[: -len("Read more")].rstrip()
        results.append({"position": len(results) + 1, "page": page, "title": r.get("title"),
                        "url": link, "display_url": r.get("display_url"), "snippet": snippet})
    print(f"page {page}: {len(data['rows'])} organic, {added} new")
    if page == 1:
        words = {w for w in QUERY.lower().split() if len(w) > 2}
        if words and not any(w in (r.get("title") or "").lower() for r in data["rows"] for w in words):
            print("WARNING: no page-1 title shares a word with the query; Bing may have served off-topic results")
    if not isinstance(data.get("next"), str) or not data["next"]:
        complete = True  # a successfully loaded page without a Next link is the real end
        break
    if page == MAX_PAGES:
        print(f"stopped at MAX_PAGES={MAX_PAGES} while Bing still had a Next link: NOT the full result set")
        break
    time.sleep(random.uniform(3, 6))  # polite pacing between pages
    url = urllib.parse.urljoin("https://www.bing.com", data["next"])  # follow Bing's own Next link

print(len(results), "unique results, complete =", complete)
print(json.dumps(results[:2] + results[-1:], indent=1, ensure_ascii=False))
print("missing url:", sum(1 for r in results if not r["url"]), "missing title:", sum(1 for r in results if not r["title"]),
      "missing snippet:", sum(1 for r in results if not r["snippet"]))
```

Real output (trimmed):

```text
page 1: 2 organic, 2 new
page 2: 6 organic, 5 new
page 3: 14 organic, 14 new
stopped at MAX_PAGES=3 while Bing still had a Next link: NOT the full result set
21 unique results, complete = False
[
 {
  "position": 1,
  "page": 1,
  "title": "These Are The Best EVs You Can Buy In 2026 - Edmunds",
  "url": "https://www.edmunds.com/car-news/best-evs-you-can-buy-in-2026.html",
  "display_url": "https://www.edmunds.com › car-news",
  "snippet": "Dec 26, 2025 · There are more EVs to pick from right now than ever before, and that can make shopping …"
 },
 ...
 {
  "position": 21,
  "page": 3,
  "title": "The 19 Best EVs Coming in 2026 - WIRED",
  "url": "https://www.wired.com/story/the-16-best-evs-coming-in-2026/",
  "display_url": "https://www.wired.com › story",
  "snippet": "Mar 21, 2026 · Without question, one of the most anticipated EVs for 2026 is Ferrari’s Luce, which isn’t just the marque’s first electric …"
 }
]
missing url: 0 missing title: 0 missing snippet: 0
```

Raise `MAX_PAGES` until the run ends with `complete = True` if you need every page. With `QUERY = "web scraping python"` and `MAX_PAGES = 6` an earlier version of this loop (same extraction and Next-link logic) returned 71 and 76 unique results on two runs, all with title and snippet.

## What works: related searches

Bing shows a "Searches you might like" block (`.b_rs`, inside an `li.b_ans` in the main column) with 8 or 10 links. It loads asynchronously, so poll for it until the link count stops changing. It was present for 5 of 6 queries tried (`best running shoes`, `how to make sourdough bread`, `cheap flights to tokyo`, `python vs javascript`, `coffee grinder`); `web scraping python` never showed it, even after scrolling. The suggestions changed between loads of the same query (8 different ones on one load, 10 on a later one). The page check is the same `open_search()` as above, so related links are never read from a previous page.

```python
import base64, binascii, json, time, urllib.parse

QUERY = "best running shoes"


class BingPageError(RuntimeError):
    pass


STATE = """JSON.stringify({url: location.href, stale: !!window.__bhStale, ready: document.readyState,
  n: document.querySelectorAll('#b_results > li.b_algo').length})"""


def open_search(url, query, tries=3, timeout=20):
    """Navigate to a Bing results URL and return only when the NEW document is that query's results page."""
    last = None
    for attempt in range(1, tries + 1):
        try:
            js("window.__bhStale = true")  # marks the current document; a freshly loaded one lacks it
        except (TimeoutError, RuntimeError) as e:
            last = f"could not mark the current page: {e}"  # never navigate unmarked: we could read the old page
            time.sleep(5 * attempt)
            continue
        try:
            goto_url(url)
        except TimeoutError as e:
            last = f"goto_url timed out: {e}"
        else:
            end = time.time() + timeout
            while time.time() < end:
                time.sleep(0.5)
                try:
                    st = json.loads(js(STATE))
                except (TimeoutError, RuntimeError, TypeError, ValueError) as e:  # page swapping mid-read
                    last = f"state read failed: {e}"
                    continue
                last = st
                if st["stale"] or st["ready"] != "complete":
                    continue
                p = urllib.parse.urlparse(st["url"])
                qs = urllib.parse.parse_qs(p.query)
                if p.hostname != "www.bing.com" or p.path != "/search":
                    break  # challenge, consent or redirect page: not results
                if (qs.get("q") or [""])[0].lower().split() != query.lower().split() or "rdr" in qs:
                    break  # wrong query, or Bing bounced the URL back to page 1
                if st["n"]:
                    return st
        time.sleep(5 * attempt)
    raise BingPageError(f"no Bing results page for {url} after {tries} tries; last state: {last}")


def related_query(href):
    """Related-search links are /ck/a redirects whose u=a1<base64url> decodes to /search?q=...&FORM=BMQS0N.
    Returns the q value, or None if the link does not decode to a search URL."""
    if not isinstance(href, str):
        return None
    parts = urllib.parse.urlparse(href)
    u = (urllib.parse.parse_qs(parts.query).get("u") or [""])[0]
    if u.startswith("a1"):
        s = u[2:]
        try:
            href = base64.urlsafe_b64decode(s + "=" * (-len(s) % 4)).decode("utf-8")
        except (binascii.Error, ValueError):
            return None
    return (urllib.parse.parse_qs(urllib.parse.urlparse(href).query).get("q") or [None])[0]


new_tab("about:blank")
open_search("https://www.bing.com/search?" + urllib.parse.urlencode({"q": QUERY}), QUERY)

related, prev = [], -1
for _ in range(20):  # the block loads asynchronously; poll up to ~10 s until the count is stable
    try:
        related = js("""(() => [...document.querySelectorAll('.b_rs a')].map(a => ({
            text: a.textContent.replace(/\\s+/g, ' ').trim(), href: a.href})))()""") or []
    except (TimeoutError, RuntimeError):
        related = []
    if related and len(related) == prev:
        break
    prev = len(related)
    time.sleep(0.5)

out = []
for i, r in enumerate(related):
    q = related_query(r.get("href"))
    out.append({"position": i + 1, "text": r.get("text"), "query": q,
                "url": "https://www.bing.com/search?" + urllib.parse.urlencode({"q": q}) if q else None})
print(len(out), "related searches")
print(json.dumps(out, indent=1, ensure_ascii=False))
```

Real output (trimmed):

```text
10 related searches
[
 {"position": 1, "text": "best running shoes for men", "query": "best running shoes for men",
  "url": "https://www.bing.com/search?q=best+running+shoes+for+men"},
 {"position": 2, "text": "best running shoes for women", "query": "best running shoes for women",
  "url": "https://www.bing.com/search?q=best+running+shoes+for+women"},
 ...
 {"position": 7, "text": "best running shoes nike", "query": "best running shoes Nike",
  "url": "https://www.bing.com/search?q=best+running+shoes+Nike"},
 ...
 {"position": 10, "text": "best running shoes for marathon", "query": "best running shoes for marathon",
  "url": "https://www.bing.com/search?q=best+running+shoes+for+marathon"}
]
```

`text` is the lower-cased display text; `query` is the decoded `q` Bing would actually run (it can differ in case). An empty list means the block did not appear for that query (it is optional on Bing's side, not a load failure: `open_search()` already confirmed the page).

## Fields

Organic results (block 1), one dict per unique result:

| Field | Source | Notes |
|---|---|---|
| `position` | running index over unique results | 1-based across all pages, after dedupe |
| `page` | loop counter | Bing page number (1 = first page) |
| `title` | `li.b_algo h2 a` text | Long titles end in `…` (Bing truncates). `None` if the row has no `h2 a` |
| `url` | `h2 a` href, `u=a1...` decoded | Raw href is `https://www.bing.com/ck/a?!&&p=...&u=a1<base64url>&ntb=1`; `msockid` param stripped. If `u` does not decode, the raw Bing redirect is kept. `None` only if the row has no link (such rows are kept, not deduped) |
| `display_url` | `li.b_algo cite` text | Breadcrumb style, e.g. `https://www.edmunds.com › car-news`, often truncated with `...` |
| `snippet` | `.b_caption p` (fallback: first `p` in the same `li`) | Often starts with a date, e.g. `Dec 26, 2025 · ` or `4 days ago · `; split on ` · ` if you need it. Usually truncated with `…`; a trailing `Read more` link text is stripped |

Run-level: `complete` is `True` only when a successfully loaded page had no Next link.

Related searches (block 2), one dict per link:

| Field | Source | Notes |
|---|---|---|
| `position` | link order in `.b_rs` | 1-based |
| `text` | `.b_rs a` text | Lower-cased display text |
| `query` | `q` of the decoded `u=a1...` | `u` decodes to `/search?q=<query>&FORM=BMQS01..08`; `None` if it does not decode |
| `url` | built from `query` | `https://www.bing.com/search?q=<query>`; `None` when `query` is `None` |

## Gotchas

- **innerText is empty for results below the fold.** Bing skips rendering off-screen `li.b_algo` items, so `innerText` returned `""` for 9 of 14 items on a page 3 while `textContent` had the text. Use `textContent` (the extractor does).
- **Do not deep-link `first=N`.** In a fresh, cookie-less session, `.../search?q=web+scraping+python&first=11` was redirected to `...&first=11&rdr=1&rdrig=...` and showed page 1 again. In a session that already had Bing cookies the same URL returned real page 2, so it is not reliable either way. Following `a.sb_pagN` worked on every run. `open_search()` treats an `rdr` URL as a failed load.
- **Offsets and page sizes are irregular.** Next links went `first=11, 25, 39` on one session and `first=7, 21, 35` (or `5, 19, 33`, or `3`/`4`) on others, with 2 to 14 organic results per page. Do not assume 10 per page; count what you get, and only stop on a missing Next link.
- **Results occasionally go off-topic.** In one browser load (a URL with `&first=1&setlang=en-US&cc=US` that got the `rdr=1` redirect), `web scraping python` returned the same unrelated "web" results the plain-HTTP path gets (WhatsApp Web, World Wide Web - Wikipedia), and all 3 pages were identical. A plain `?q=` URL fixed it. A nonsense query (`qzxjvkwpq zzxqvbn flarbnoqtz`) still returned 10 (on a later load, 5) `li.b_algo` and no Next link rather than an empty page; block 1 ended that run with `complete = True` and printed its `WARNING`. Block 1 prints the `WARNING` when no page-1 title shares a word with the query; reload once with a plain `?q=` URL before trusting such a run.
- **Zero results raises.** `open_search()` requires at least one `li.b_algo`, so a query Bing answers with no organic results raises `BingPageError` after 3 tries. No such page was seen in this run (see the nonsense query above), so there is no tested "no results" selector.
- **Only `li.b_algo` are organic.** Other `#b_results > li` seen: `b_ans` (answer blocks, including related searches), `b_msg` ("Some results have been removed"), `b_ad` (ads), `b_pag` (pagination). They are skipped by the selector.
- **Right rail.** For most queries `#b_context` held a "Deep dive into <query>" Copilot block, not related searches. The related searches were always in the main column.
- **No consent banner, CAPTCHA or region redirect** appeared in any of the roughly 50 browser page loads in this run; the host stayed `www.bing.com`. If one does appear, `open_search()` raises with the last URL it saw instead of parsing it.
- **`js()` can time out right after a load.** One poll about 1.5 s after a results page reached `complete` raised `RuntimeError: Runtime.evaluate timed out` (the underlying `_IPCResponseTimeout` is a `TimeoutError` subclass). `open_search()` keeps polling through it and block 1 reads the page with `js_retry()`.
- **Background-tab scroll can time out.** `scroll()` on the attached tab timed out once (`Input.dispatchMouseEvent timed out after 5s`). The scripts above do not scroll and never bring the tab to the foreground.

## What does not work

- **Official API:** retired August 11, 2025 per Microsoft's lifecycle page; the v7 endpoint returns `401 PermissionDenied` without a key and no new keys are issued.
- **Plain HTTP pagination:** cookie-less `GET /search` (User-Agent `Mozilla/5.0`) returned page 1 for every Next link and for `first=11`, three queries tested.
- **Plain HTTP relevance:** `web scraping python` and `scrapy tutorial` returned unrelated results over plain HTTP (10 results each, HTTP 200). `electric cars 2026`, `best running shoes` and `sourdough bread recipe` matched the browser. A truthful client User-Agent (`curl/8.7.1`, `Python-urllib/3.12`) got the same 200 and 10 results as `Mozilla/5.0`, and the same unrelated results for `web scraping python`, so the User-Agent is not the cause. The failure is silent, so plain HTTP is not safe for arbitrary queries.
- **`format=rss`:** `/search?q=...&format=rss` returns `text/xml` with 10 `<item>`s (title, link, description, pubDate) but shows the same unrelated results as plain HTTP for `web scraping python`, and its copyright element restricts use to "rendering Bing results within an RSS aggregator for your personal, non-commercial use".
- **Related searches over plain HTTP:** the cookie-less HTML for `best running shoes` (10 good organic results) had no `.b_rs` block; it only appears after the page's scripts run in a browser.
