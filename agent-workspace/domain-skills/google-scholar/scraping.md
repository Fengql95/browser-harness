# Google Scholar — Search Results Scraping

Field-tested against `scholar.google.com` on 2026-10-23 using a Browser Use cloud browser and plain HTTP (`urllib`).

## Quick summary

- **Read this first:** `https://scholar.google.com/robots.txt` has `User-agent: *` / `Disallow: /scholar`, and Scholar can put a CAPTCHA wall (`google.com/sorry/`) in front of automated traffic. This skill is for a person-sized lookup (a few pages, tens of results). For bulk literature data use the `openalex` or `crossref` skills instead; they have real APIs.
- **There is no JSON.** Results are server-rendered HTML: no JSON-LD, no `__NEXT_DATA__`, no `application/json` blocks were found in the page. You parse the HTML (or the DOM in a browser).
- **Both paths worked, and both can be walled.** Plain `GET /scholar?q=...` with `User-Agent: Mozilla/5.0` returned 10 parsed results per page on every request (five 3-page runs, 20-40 s apart). In the browser, one cloud browser session was sent to `google.com/sorry/` on its very first request, homepage included; two later fresh cloud browsers loaded the homepage, search pages and `start=10,20` with no wall. Whether you get walled depends on the IP you come from, so check every page and stop on the first wall.
- **10 results per page, `start=0,10,20,...`.** Seven 3-page runs (HTTP and browser) had between 0 and 4 repeated rows out of 30, so dedupe on the result id (`data-cid`).
- **When you hit the wall, stop.** Do not retry in a loop, rotate IPs or solve the CAPTCHA. Back off and switch to an API source.

## What works: paced HTTP search, 3 pages, stdlib only

Runs as plain `python3` (no browser, no extra packages). The query is URL-encoded with `urlencode`, `as_ylo` is the "since year" filter from Scholar's advanced search form.

```python
import json, random, re, time, urllib.error, urllib.parse, urllib.request
from html.parser import HTMLParser

BASE = "https://scholar.google.com"
HEADERS = {"User-Agent": "Mozilla/5.0", "Accept-Language": "en-US,en;q=0.9"}
VOID = {"br", "img", "input", "meta", "link", "hr", "wbr", "path"}


class ScholarBlocked(Exception):
    pass


class _Block(HTMLParser):
    """Text per CSS class plus every <a> (with its ancestor classes) inside one result."""

    def __init__(self):
        super().__init__(convert_charrefs=True)
        self.stack, self.text, self.links = [], {}, []

    def handle_starttag(self, tag, attrs):
        a = dict(attrs)
        cls = (a.get("class") or "").split()
        if tag == "br":
            self.handle_data(" ")
        if tag in VOID:
            return
        self.stack.append((tag, cls))
        if tag == "a":
            self.links.append({"href": a.get("href") or "", "cls": cls, "text": "",
                               "inside": {c for _, cs in self.stack for c in cs}})

    def handle_endtag(self, tag):
        for i in range(len(self.stack) - 1, -1, -1):
            if self.stack[i][0] == tag:
                del self.stack[i:]
                break

    def handle_data(self, d):
        for _, cls in self.stack:
            for c in cls:
                self.text[c] = self.text.get(c, "") + d
        if self.links and any(t == "a" for t, _ in self.stack):
            self.links[-1]["text"] += d


def _clean(s):
    return re.sub(r"\s+", " ", (s or "").replace("\xa0", " ")).strip()


def parse_results(html):
    starts = [m.start() for m in re.finditer(r'<div class="gs_r gs_or gs_scl"', html)]
    rows = []
    for i, s in enumerate(starts):
        chunk = html[s: starts[i + 1] if i + 1 < len(starts) else len(html)]
        p = _Block()
        p.feed(chunk)
        t = p.text
        first = lambda pred: next((l for l in p.links if pred(l)), None)
        title_a = first(lambda l: "gs_rt" in l["inside"])
        side_a = first(lambda l: "gs_or_ggsm" in l["inside"])
        cited_a = first(lambda l: "gs_or_cited" in l["cls"])
        vers_a = first(lambda l: "cluster=" in l["href"] and "versions" in l["text"])
        rel_a = first(lambda l: "q=related:" in l["href"])
        tag = _clean(t.get("gs_ct1")) or None            # e.g. [PDF], [HTML], [BOOK], [PS]
        if title_a:
            title = _clean(title_a["text"])
        else:                                            # defensive: keep the bare h3 text if there is no link
            title = _clean(t.get("gs_rt", "").replace(t.get("gs_ctc", ""), "").replace(t.get("gs_ctu", ""), ""))
        byline = _clean(t.get("gs_a"))                   # "authors - venue, year - host"
        parts = [x.strip() for x in byline.split(" - ")]
        venue_year = parts[1] if len(parts) >= 2 else ""
        year = re.search(r"\b(?:19|20)\d{2}\b", venue_year)
        cid = re.search(r'data-cid="([^"]+)"', chunk)
        rows.append({
            "id": cid.group(1) if cid else None,
            "tag": tag,
            "title": title,
            "url": urllib.parse.urljoin(BASE, title_a["href"]) if title_a and title_a["href"] else None,
            "authors": parts[0] or None,
            "venue_year": venue_year or None,
            "year": int(year.group(0)) if year else None,
            "source": parts[2] if len(parts) >= 3 else None,
            "snippet": _clean(t.get("gs_rs")) or None,
            "cited_by": int(re.sub(r"\D", "", cited_a["text"]) or 0) if cited_a else 0,
            "cited_by_url": urllib.parse.urljoin(BASE, cited_a["href"]) if cited_a else None,
            "versions_url": urllib.parse.urljoin(BASE, vers_a["href"]) if vers_a else None,
            "related_url": urllib.parse.urljoin(BASE, rel_a["href"]) if rel_a else None,
            "fulltext_url": side_a["href"] if side_a else None,
            "fulltext_label": _clean(side_a["text"]) if side_a else None,
        })
    return rows


def fetch_page(query, start=0, year_from=None):
    params = {"q": query, "hl": "en", "start": start}
    if year_from:
        params["as_ylo"] = year_from
    url = f"{BASE}/scholar?{urllib.parse.urlencode(params)}"
    req = urllib.request.Request(url, headers=HEADERS)
    try:
        with urllib.request.urlopen(req, timeout=30) as r:
            final_url, html = r.geturl(), r.read().decode("utf-8", "replace")
    except urllib.error.HTTPError as e:
        raise ScholarBlocked(f"HTTP {e.code} at {e.geturl()}") from e
    if "/sorry/" in final_url or "unusual traffic" in html:
        raise ScholarBlocked(f"CAPTCHA wall at {final_url}")
    total = re.search(r'<div id="gs_ab_md"><div class="gs_ab_mdw">(?:About )?([\d,]+) result', html)
    has_next = re.search(r'<a [^>]*><span class="gs_ico gs_ico_nav_next">', html) is not None
    return {"url": url, "total": int(total.group(1).replace(",", "")) if total else None,
            "has_next": has_next, "rows": parse_results(html)}


def scholar_search(query, max_pages=3, year_from=None, delay=(20, 40)):
    results, seen, pages, total = [], set(), [], None
    for page in range(max_pages):
        if page:
            time.sleep(random.uniform(*delay))           # polite pacing between pages
        try:
            res = fetch_page(query, start=page * 10, year_from=year_from)
        except ScholarBlocked as e:
            print("STOP:", e)                            # do not retry or solve it; back off
            break
        total = total or res["total"]                    # only page 1 header parses
        fresh = [r for r in res["rows"] if r["id"] not in seen]
        seen.update(r["id"] for r in fresh)
        results.extend(fresh)
        pages.append({"start": page * 10, "rows": len(res["rows"]), "new": len(fresh), "has_next": res["has_next"]})
        if not res["rows"] or not res["has_next"]:
            break
    return results, pages, total


rows, pages, total = scholar_search('web agents "tool use"', max_pages=3, year_from=2024)
print("total:", total, json.dumps(pages))
print(len(rows), "unique rows")
print("years:", sorted({r["year"] for r in rows if r["year"]}))
print("with cited_by>0:", sum(r["cited_by"] > 0 for r in rows), "with fulltext:", sum(bool(r["fulltext_url"]) for r in rows))
print("tags:", sorted({r["tag"] for r in rows if r["tag"]}))
for r in rows[:3]:
    print(json.dumps({k: v for k, v in r.items() if k != "authors"}, ensure_ascii=False))
```

Real output (second row, `related_url` and `versions_url` trimmed; `authors` left out on purpose, it is a string of real names):

```text
total: 13700 [{"start": 0, "rows": 10, "new": 10, "has_next": true}, {"start": 10, "rows": 10, "new": 10, "has_next": true}, {"start": 20, "rows": 10, "new": 10, "has_next": true}]
30 unique rows
years: [2024, 2025, 2026]
with cited_by>0: 30 with fulltext: 30
tags: []
{"id": "3NERNDWIQgkJ", "tag": null, "title": "The Tool Illusion: Rethinking Tool Use in Web Agents", "url": "https://arxiv.org/abs/2604.03465", "venue_year": "arXiv preprint arXiv …, 2026", "year": 2026, "source": "arxiv.org", "snippet": "… tool use in web agents through an extensive and carefully controlled study across diverse tool sources, backbone models, tool-use … and inspires future research on tool-use web agents. …", "cited_by": 3, "cited_by_url": "https://scholar.google.com/scholar?cites=667245456892482012&as_sdt=2005&sciodt=0,5&hl=en", "fulltext_url": "https://arxiv.org/pdf/2604.03465", "fulltext_label": "[PDF] arxiv.org"}
{"id": "jed5XCR57OkJ", "tag": null, "title": "Verltool: Towards holistic agentic reinforcement learning with tool use", "url": "https://arxiv.org/abs/2509.01055", "venue_year": "arXiv preprint arXiv …, 2025", "year": 2025, "source": "arxiv.org", "snippet": "… tool use with RLVR, giving rise to a new paradigm we term ARLT—Agentic Reinforcement Learning with Tool use. … We implement a Web Search tool using Google Search API through …", "cited_by": 105, "cited_by_url": "https://scholar.google.com/scholar?cites=16855980702418200461&as_sdt=2005&sciodt=0,5&hl=en", "fulltext_url": "https://arxiv.org/pdf/2509.01055?", "fulltext_label": "[PDF] arxiv.org"}
```

## What works: the same search in a browser, 3 pages

Use this when you already have a browser session (browser-harness helpers `new_tab`, `goto_url`, `wait_for_load`, `js`). It reads the rendered DOM with the same class names, so the fields match the HTTP version. It checks every page for the wall and stops on it.

```python
import json, random, re, time, urllib.parse

EXTRACT = r"""
(() => {
  const clean = s => (s || '').replace(/\s+/g, ' ').trim();
  const abs = h => h ? new URL(h, location.href).href : null;
  const rows = [...document.querySelectorAll('div.gs_r.gs_or.gs_scl')].map(el => {
    const h3 = el.querySelector('h3.gs_rt');
    const a = h3 && h3.querySelector('a');
    const tag = h3 && h3.querySelector('.gs_ct1');
    const parts = clean(el.querySelector('.gs_a')?.innerText).split(' - ').map(s => s.trim());
    const cited = el.querySelector('a.gs_or_cited') ||
      [...el.querySelectorAll('.gs_fl a')].find(x => /^Cited by \d+/.test(x.innerText));
    const vers = [...el.querySelectorAll('a[href*="cluster="]')].find(x => /versions/.test(x.innerText));
    const rel = el.querySelector('a[href*="q=related:"]');
    const side = el.querySelector('.gs_or_ggsm a');
    return {
      id: el.getAttribute('data-cid'),
      tag: tag ? clean(tag.innerText) : null,
      title: a ? clean(a.innerText) : clean((h3?.innerText || '').replace(/^\[[A-Z]+\]\s*/, '')),
      url: a ? abs(a.getAttribute('href')) : null,
      authors: parts[0] || null,
      venue_year: parts[1] || null,
      source: parts[2] || null,
      snippet: clean(el.querySelector('.gs_rs')?.innerText) || null,
      cited_by: cited ? parseInt(cited.innerText.replace(/\D/g, '') || '0', 10) : 0,
      cited_by_url: cited ? abs(cited.getAttribute('href')) : null,
      versions_url: vers ? abs(vers.getAttribute('href')) : null,
      related_url: rel ? abs(rel.getAttribute('href')) : null,
      fulltext_url: side ? side.href : null,
      fulltext_label: side ? clean(side.innerText) : null,
    };
  });
  const total = (document.querySelector('#gs_ab_md')?.innerText || '').match(/([\d,]+) results?/);
  return JSON.stringify({
    url: location.href,
    blocked: location.href.includes('/sorry/') || document.body.innerText.includes('unusual traffic'),
    total: total ? parseInt(total[1].replace(/,/g, ''), 10) : null,
    has_next: !!document.querySelector('.gs_ico_nav_next'),
    rows,
  });
})()
"""


def scholar_search_browser(query, max_pages=3, year_from=None, delay=(20, 40)):
    results, seen, pages, total = [], set(), [], None
    for page in range(max_pages):
        params = {"q": query, "hl": "en", "start": page * 10}
        if year_from:
            params["as_ylo"] = year_from
        url = "https://scholar.google.com/scholar?" + urllib.parse.urlencode(params)
        if page == 0:
            new_tab(url)
        else:
            time.sleep(random.uniform(*delay))           # polite pacing between pages
            goto_url(url)
        wait_for_load()
        res = json.loads(js(EXTRACT) or "{}")
        if res.get("blocked"):
            print("STOP: CAPTCHA wall at", res.get("url"))   # do not solve it; back off
            break
        total = total or res.get("total")
        for r in res.get("rows", []):
            m = re.search(r"\b(?:19|20)\d{2}\b", r["venue_year"] or "")
            r["year"] = int(m.group(0)) if m else None
        fresh = [r for r in res.get("rows", []) if r["id"] not in seen]
        seen.update(r["id"] for r in fresh)
        results.extend(fresh)
        pages.append({"start": page * 10, "rows": len(res.get("rows", [])), "new": len(fresh), "has_next": res.get("has_next")})
        if not res.get("rows") or not res.get("has_next"):
            break
    return results, pages, total


rows, pages, total = scholar_search_browser('web agents "tool use"', max_pages=3, year_from=2024)
print("total:", total, json.dumps(pages))
print(len(rows), "unique rows")
print("years:", sorted({r["year"] for r in rows if r["year"]}))
print("with cited_by>0:", sum(r["cited_by"] > 0 for r in rows), "with fulltext:", sum(bool(r["fulltext_url"]) for r in rows))
print("tags:", sorted({r["tag"] for r in rows if r["tag"]}))
for r in rows[:2]:
    print(json.dumps({k: v for k, v in r.items() if k != "authors"}, ensure_ascii=False))
```

Real output from a fresh cloud browser (`related_url`/`versions_url` and second row trimmed, `authors` left out):

```text
total: 6820 [{"start": 0, "rows": 10, "new": 10, "has_next": true}, {"start": 10, "rows": 10, "new": 8, "has_next": true}, {"start": 20, "rows": 10, "new": 10, "has_next": true}]
28 unique rows
years: [2025, 2026]
with cited_by>0: 27 with fulltext: 27
tags: []
{"id": "SX73WcmwGNsJ", "tag": null, "title": "Walt: Web agents that learn tools", "url": "https://proceedings.iclr.cc/paper_files/paper/2026/hash/5b175f9e93873e3a10a6ce43dbb82e05-Abstract-Conference.html", "venue_year": "International …, 2026", "source": "proceedings.iclr.cc", "snippet": "… For the task “find the cheapest blue kayak,” traditional web agents execute a lengthy … vation space), also outperforms strong baselines based on test-time search and tool use by …", "cited_by": 20, "cited_by_url": "https://scholar.google.com/scholar?cites=15787562872591580745&as_sdt=2005&sciodt=0,5&hl=en", "fulltext_url": "https://proceedings.iclr.cc/paper_files/paper/2026/file/5b175f9e93873e3a10a6ce43dbb82e05-Paper-Conference.pdf", "fulltext_label": "[PDF] iclr.cc", "year": 2026}
```

## Fields

| Field | Source in the result block | Notes |
|---|---|---|
| `id` | `data-cid` on `div.gs_r.gs_or.gs_scl` | Stable per paper cluster, use it to dedupe across pages |
| `tag` | `span.gs_ct1` inside `h3.gs_rt` | Seen: `[PDF]`, `[HTML]`, `[BOOK]`, `[PS]`; `None` for normal hits |
| `title` | text of the `h3.gs_rt a` link | Every result seen in this run had the link; the code falls back to the bare `h3` text without it |
| `url` | `h3.gs_rt a[href]` | Publisher or repository page; for `[PDF]` rows it is the PDF itself |
| `authors` | `div.gs_a`, part before the first ` - ` | Abbreviated initials, often truncated with `…` |
| `venue_year` | `div.gs_a`, middle part | Often truncated, e.g. `International …, 2026`, `arXiv preprint arXiv …, 2025` |
| `year` | 4-digit year in `venue_year` | Can be `None` |
| `source` | `div.gs_a`, last part | Host or publisher, e.g. `arxiv.org`, `proceedings.iclr.cc`, `dl.acm.org` |
| `snippet` | `div.gs_rs` | Two lines of matched text, starts and ends with `…` |
| `cited_by` | `a.gs_or_cited` text `Cited by N` | `0` when the link is absent |
| `cited_by_url` | `a.gs_or_cited[href]` | A `/scholar?cites=<cluster>` results page |
| `versions_url` | `a[href*="cluster="]` with text `All N versions` | Absent for single-version papers |
| `related_url` | `a[href*="q=related:"]` | |
| `fulltext_url`, `fulltext_label` | `div.gs_or_ggsm a` (right-hand side link) | e.g. `[PDF] arxiv.org`; absent for many paywalled papers |
| `total` | `#gs_ab_md` on page 1 | `About 13,700 results` -> `13700`; an estimate |

## Gotchas

- **robots.txt and the wall come first.** `/scholar` is disallowed for all user agents. Keep runs small, pace them (20-40 s between pages was used here), and stop on the first block. There is no official Scholar API.
- **The wall depends on the IP.** One cloud browser session landed on `https://www.google.com/sorry/index?continue=https://scholar.google.com/...` with the text "Our systems have detected unusual traffic from your computer network" on its first request, including for the bare homepage. Two other fresh cloud browsers the same day were never walled. A browser is not a way around the wall; if you see it, stop and use an API source.
- **Duplicates across pages.** 3-page runs returned 30 rows but 27, 29, 26, 30 and 27 unique ids over HTTP and 30 and 28 in the browser. Always dedupe on `id`.
- **`total` only on page 1, and it is rough.** The HTTP regex returned `None` on pages 2 and 3, so the code keeps the first page's value. The same query reported 8,410, 8,970, 9,930, 13,700 and 15,600 (HTTP) and 4,300 and 6,820 (browser) results within an hour, and the first result differed between the HTTP and browser runs. Do not use it as a denominator.
- **Truncated metadata.** Authors and venue are shortened with `…` by Scholar itself. Follow `url`, or look the title up in OpenAlex/Crossref, if you need full author lists, DOI or the full venue name.
- **No consent banner** was served to plain HTTP or to the cloud browser (no "Before you continue" text). `hl=en` keeps the English link texts (`Cited by`, `All N versions`) that the parser matches on.
- **Selector drift.** Both versions rely on Scholar's long-lived class names (`gs_r gs_or gs_scl`, `gs_rt`, `gs_ct1`, `gs_a`, `gs_rs`, `gs_or_cited`, `gs_or_ggsm`, `gs_ico_nav_next`). Result data has no ARIA labels, so classes are the only hook. If you get 0 rows on a page that is not walled, check those class names first.
- **Trailing `?` on some full-text links.** `fulltext_url` can end in a bare `?` (e.g. `https://arxiv.org/pdf/2509.01055?` in the HTTP output above). Strip it if you compare URLs.

## What does not work

- **Assuming you will not be walled.** One of three cloud browser sessions was redirected to the `google.com/sorry/` CAPTCHA on every navigation, homepage included. If that happens to you, stop. Do not solve or auto-solve it to keep scraping.
- **Structured data.** No JSON-LD (0 `application/ld+json` scripts in the browser), embedded JSON, or public JSON endpoint for search results.
- **Bulk crawling.** Deep pagination, many queries in a row, or crawling `cited_by_url` / `versions_url` for every row was not attempted, because `/scholar` is disallowed in robots.txt and the wall can trigger on low volume from some networks. Use the `openalex` or `crossref` skills for anything bigger than a handful of pages.
