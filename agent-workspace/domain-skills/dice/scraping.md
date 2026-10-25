# Dice — Tech Job Search Scraping

Field-tested against www.dice.com on 2026-10-23 using a Browser Use cloud browser and plain HTTP (Python urllib).

robots.txt (fetched 2026-10-23) disallows `/jobs?q*`, `/jobsearch/` and `/job` for `User-agent: *` but allows `/jobs` and `/job-detail`, so this skill only requests the `/jobs/q-<keyword>-l-<location>-jobs` search pages (the same shape Dice lists in its own `sitemap_ts.xml` / `sitemap_tsl1.xml`) and `/job-detail/<guid>`.

## Quick summary

- Search result pages are server-rendered Next.js (App Router). Every page embeds the full result set as JSON inside `self.__next_f.push([1,"..."])` script strings, under a `"jobList":` key: `data` (20 jobs) + `meta` (`currentPage`, `pageCount`, `totalResults`, facets).
- Plain HTTP works: a truthful custom User-Agent got HTTP 200 and the full JSON on every request in this run (about 80 page requests, mostly at a 2 s pace, no 429, no challenge, no consent banner).
- Page size is fixed at 20. Pages past 25 are clamped: `?page=26` and `?page=46` both returned `currentPage: 25`. So one search yields at most 500 jobs even when `pageCount` says 46.
- Relevance order is not stable between requests, so deep pagination repeats and skips some jobs (78-99 unique per 100 slots measured). De-dup on `guid`.
- `/job-detail/<guid>` pages carry a schema.org `JobPosting` JSON-LD block (full description, `validThrough`, postal address).
- The browser path works too (same JSON, read from the inline scripts), but it is not needed for logged-out search.

## What works

### 1. Search + paginate over plain HTTP (main method)

Self-contained, standard library only. Builds the public `/jobs/q-...-jobs` path, follows `?page=N`, stops at `pageCount` or at the 25-page clamp, de-duplicates by `guid`, and raises (no silent partial data) on blocks or layout changes.

```python
import json, re, time, urllib.error, urllib.parse, urllib.request

UA = "dice-jobs-research/1.0 (python-urllib; personal job search script)"
BASE = "https://www.dice.com"
MAX_PAGE = 25        # pages past 25 come back as page 25
PAUSE_S = 2.0        # polite gap between page requests

def search_url(keyword=None, location=None, page=1, filters=None):
    """Public search path /jobs/q-<kw>-l-<loc>-jobs. Hyphens are path separators
    there, so hyphens inside the keyword/location are replaced with spaces."""
    parts = []
    if keyword:
        parts.append("q-" + urllib.parse.quote_plus(keyword.replace("-", " ").strip()))
    if location:
        parts.append("l-" + urllib.parse.quote_plus(location.replace("-", " ").strip()))
    if not parts:
        raise ValueError("need a keyword or a location")
    params = dict(filters or {})
    if page > 1:
        params["page"] = page
    url = f"{BASE}/jobs/{'-'.join(parts)}-jobs"
    return url + ("?" + urllib.parse.urlencode(params) if params else "")

def fetch(url, attempts=4):
    """GET with capped backoff (5+15+40 s = 60 s max), no sleep after the last try."""
    delays = [5, 15, 40]
    for i in range(attempts):
        try:
            req = urllib.request.Request(url, headers={"User-Agent": UA, "Accept": "text/html"})
            with urllib.request.urlopen(req, timeout=30) as r:
                if r.geturl() != url:
                    raise RuntimeError(f"redirected to {r.geturl()}")
                return r.read().decode("utf-8")
        except urllib.error.HTTPError as e:
            if e.code not in (429, 500, 502, 503, 504) or i == attempts - 1:
                raise
        except urllib.error.URLError:
            if i == attempts - 1:
                raise
        time.sleep(delays[i])

def parse_joblist(html):
    """Concatenate the RSC strings and decode the object after "jobList":."""
    chunks = re.findall(r'self\.__next_f\.push\(\[1,(".*?")\]\)</script>', html, re.S)
    flight = "".join(json.loads(c) for c in chunks)
    i = flight.find('"jobList":')
    if i < 0:
        raise RuntimeError("no jobList in page (layout change or block page) - stop and inspect")
    return json.JSONDecoder().raw_decode(flight, i + len('"jobList":'))[0]

def normalize(j):
    loc = j.get("jobLocation") or {}
    return {
        "title": j.get("title"),
        "company": j.get("companyName"),
        "location": loc.get("displayName"),
        "workplace": ", ".join(j.get("workplaceTypes") or []),
        "employment_type": j.get("employmentType"),
        "posted": j.get("postedDate"),
        "salary": j.get("salary"),
        "url": j.get("detailsPageUrl"),
        "guid": j.get("guid"),
    }

def search_dice(keyword=None, location=None, max_pages=3, filters=None):
    jobs, seen, total, pages = [], set(), None, None
    for page in range(1, min(max_pages, MAX_PAGE) + 1):
        if page > 1:
            time.sleep(PAUSE_S)
        jl = parse_joblist(fetch(search_url(keyword, location, page, filters)))
        meta = jl.get("meta") or {}
        total, pages = meta.get("totalResults"), meta.get("pageCount") or 0
        if meta.get("currentPage") != page:      # clamped past the last servable page
            break
        for j in jl.get("data") or []:
            if j.get("guid") and j["guid"] not in seen:
                seen.add(j["guid"])
                jobs.append(normalize(j))
        if page >= pages:
            break
    return jobs, total, pages

jobs, total, pages = search_dice("Python Developer", "Houston, TX", max_pages=3)
print(f"total={total} pageCount={pages} collected={len(jobs)}")
for r in jobs[:3]:
    print(json.dumps(r))

# Narrowing with Dice's own filter params (values come from meta.facetQueryResults):
remote, rtotal, _ = search_dice("DevOps Engineer", max_pages=2,
                                filters={"filters.workplaceTypes": "Remote"})
print(f"remote devops total={rtotal} collected={len(remote)}")
```

Output observed (trimmed):

```
total=132 pageCount=7 collected=60
{"title": "Python Developer - Houston, TX (Locals only)", "company": "Source Mantra Inc", "location": "Houston, Texas, USA", "workplace": "On-Site", "employment_type": "Third Party, Contract", "posted": "2026-10-23T19:05:49Z", "salary": "$$65 - $70", "url": "https://www.dice.com/job-detail/7a92a6a3-2b6f-49be-9f2a-c6737ea95e2e", "guid": "7a92a6a3-2b6f-49be-9f2a-c6737ea95e2e"}
{"title": "Python Developer", "company": "Mindlance", "location": "Houston, Texas, USA", "workplace": "On-Site", "employment_type": "Contract", "posted": "2026-10-15T21:57:49Z", "salary": "50 - 58", ...}
{"title": "Python Developer - AWS, AI, LLM -  ZL", "company": "SES", "location": null, "workplace": "Remote", "employment_type": "Full-time", ...}
remote devops total=239 collected=40
```

With `max_pages=30` the same Houston search returned all 132 jobs over 7 pages and stopped at `pageCount`.

URL shapes that returned 200 with a `jobList`:

| Shape | Example |
|---|---|
| keyword | `/jobs/q-Rust+Engineer-jobs` (works for any keyword, not only sitemap ones) |
| keyword + location | `/jobs/q-Python+Developer-l-Houston%2C+TX-jobs` |
| location only | `/jobs/l-Austin%2C+TX-jobs` |
| page N | append `?page=2` |
| filters | `?filters.workplaceTypes=Remote`, `?filters.postedDate=ONE`, `?filters.employmentType=FULLTIME` (combine with `&page=N`) |

### 2. Same data in the browser (when you are already driving a tab)

The in-memory `self.__next_f` array is emptied after hydration (length 0 observed), but the inline `<script>` tags stay in the DOM, so read those.

```python
import json, time, urllib.parse

def search_url(keyword=None, location=None, page=1):
    parts = []
    if keyword:
        parts.append("q-" + urllib.parse.quote_plus(keyword.replace("-", " ").strip()))
    if location:
        parts.append("l-" + urllib.parse.quote_plus(location.replace("-", " ").strip()))
    url = "https://www.dice.com/jobs/" + "-".join(parts) + "-jobs"
    return url if page == 1 else f"{url}?page={page}"

READ_JOBLIST = r"""(()=>{
  const pre = 'self.__next_f.push([1,', parts = [];
  for (const s of document.querySelectorAll('script:not([src])')) {
    const t = s.textContent;
    if (!t.startsWith(pre)) continue;
    try { parts.push(JSON.parse(t.slice(pre.length, t.lastIndexOf(']')))); } catch (e) {}
  }
  const f = parts.join(''), i = f.indexOf('"jobList":');
  return i < 0 ? null : f.slice(i + 10);
})()"""

def open_page(url, first=False):
    new_tab(url) if first else goto_url(url)
    wait_for_load()
    for _ in range(20):                      # confirm we are on the requested page
        if js("location.href") == url:
            return
        time.sleep(0.5)
    raise RuntimeError(f"expected {url}, got {js('location.href')}")

def browser_search(keyword=None, location=None, max_pages=2):
    rows, seen, meta = [], set(), {}
    for page in range(1, min(max_pages, 25) + 1):
        if page > 1:
            time.sleep(2)
        open_page(search_url(keyword, location, page), first=(page == 1))
        raw = js(READ_JOBLIST)
        if not raw:
            raise RuntimeError("no jobList on page - stop and inspect (layout change or block page)")
        jl = json.JSONDecoder().raw_decode(raw)[0]
        meta = jl.get("meta") or {}
        if meta.get("currentPage") != page:
            break
        for j in jl.get("data") or []:
            if j.get("guid") in seen:
                continue
            seen.add(j.get("guid"))
            loc = j.get("jobLocation") or {}
            rows.append({"title": j.get("title"), "company": j.get("companyName"),
                         "location": loc.get("displayName"), "employment_type": j.get("employmentType"),
                         "posted": j.get("postedDate"), "url": j.get("detailsPageUrl")})
        if page >= (meta.get("pageCount") or 0):
            break
    return rows, meta.get("totalResults")

rows, total = browser_search("DevOps Engineer", "Remote", max_pages=2)
print(f"total={total} collected={len(rows)}")
for r in rows[:3]:
    print(json.dumps(r))
```

Output observed (trimmed):

```
total=1585 collected=38
{"title": "DevOps Engineer", "company": "Dexian DISYS", "location": "Charlotte, North Carolina, USA", "employment_type": "Contract", "posted": "2026-10-15T20:11:32Z", "url": "https://www.dice.com/job-detail/be07b58c-3595-4790-a5c4-fed0892f9c8c"}
{"title": "DevOps Engineer", "company": "TechNix LLC", "location": "Raleigh, North Carolina, USA", "employment_type": "Contract, Third Party", ...}
```

Not 40: across two runs this collected 38 and then 28 unique jobs, because the relevance order reshuffles between requests and page 2 repeats some page-1 jobs. See "Unstable order" under Gotchas.

DOM fallback if the JSON ever disappears: each result is `[data-testid="job-card"][role="article"]` inside `role="list"` with `aria-label="Job search results"`; the card link `a[data-testid="job-search-job-card-link"]` has `href="/job-detail/<guid>"` and `aria-label="View Details for <title> (<id>)"`. Card text only has relative dates ("Yesterday", "2d ago"), so prefer the JSON.

### 3. Job detail via JSON-LD (description, expiry, postal address)

```python
import json, re, urllib.request

UA = "dice-jobs-research/1.0 (python-urllib; personal job search script)"

def job_detail(detail_url):
    req = urllib.request.Request(detail_url, headers={"User-Agent": UA, "Accept": "text/html"})
    with urllib.request.urlopen(req, timeout=30) as r:
        if r.geturl() != detail_url:
            raise RuntimeError(f"redirected to {r.geturl()}")
        html = r.read().decode("utf-8")
    for block in re.findall(r'<script[^>]*application/ld\+json[^>]*>(.*?)</script>', html, re.S):
        d = json.loads(block)
        if isinstance(d, dict) and d.get("@type") == "JobPosting":
            return d
    return None   # expired or removed postings may not carry JSON-LD

# Take a fresh URL from a search page so this does not depend on one posting staying live.
search = "https://www.dice.com/jobs/q-Python+Developer-jobs"
with urllib.request.urlopen(urllib.request.Request(search, headers={"User-Agent": UA}), timeout=30) as r:
    first = re.search(r'href="(/job-detail/[0-9a-f-]{36})"', r.read().decode("utf-8"))
if not first:
    raise RuntimeError("no job-detail link on the search page")
jp = job_detail("https://www.dice.com" + first.group(1))
if jp:
    addr = (jp.get("jobLocation") or {}).get("address") or {}
    print(jp["title"], "|", (jp.get("hiringOrganization") or {}).get("name"), "|", jp.get("employmentType"),
          "|", jp.get("datePosted"), "|", jp.get("validThrough"), "|", addr.get("addressLocality"), addr.get("addressRegion"))
    print(len(jp.get("description") or ""), "chars of description HTML")
else:
    print("no JobPosting JSON-LD on that page")
```

Output observed (trimmed):

```
Equity Derivatives Python Developer - New York - COntract | SANS | CONTRACTOR | 2026-10-22T21:49:51.000Z | 2026-11-23T18:06:44.000Z | New York NY
```

JSON-LD keys seen: `title, datePosted, validThrough, employmentType, hiringOrganization{name,sameAs,logo}, jobLocation{address{addressLocality,addressRegion,postalCode,addressCountry}}, baseSalary, description, identifier, applicantLocationRequirements, url`. Note `employmentType` here is the schema.org enum (`CONTRACTOR`), not the search-card label (`Contract`).

## Fields (search `jobList.data[]`)

| Field | Example | Notes |
|---|---|---|
| `title` | `"Python Developer"` | |
| `companyName` | `"Mindlance"` | |
| `jobLocation.displayName` | `"Houston, Texas, USA"` | `jobLocation` is `null` for many remote jobs; also has `city`, `state`, `region`, `country` |
| `workplaceTypes` | `["Remote", "On-Site"]` | list: `On-Site`, `Remote`, `Hybrid` |
| `isRemote` | `false` | |
| `employmentType` | `"Third Party, Contract"` | comma-joined labels: `Full-time`, `Part-time`, `Contract`, `Third Party` |
| `employerType` | `"Recruiter"` | `Direct Hire`, `Recruiter`, `Other` |
| `postedDate` | `"2026-10-23T19:05:49Z"` | ISO 8601 UTC; `modifiedDate` also present |
| `salary` | `"Depends on Experience"` | free text, key missing on some jobs |
| `detailsPageUrl` | `"https://www.dice.com/job-detail/<guid>"` | absolute URL |
| `guid` / `id` | UUID / 32-hex | `guid` is the detail-page id; use it to de-dup |
| `summary` | first ~500 chars of description | |
| `easyApply`, `workFromHomeAvailability`, `companyProfileId`, `companyLogoUrl`, `score` | | |

`meta` holds `currentPage`, `pageSize` (20), `pageCount`, `totalResults`, `searchedLocation` (resolved city, lat/long, 30 mi radius), `sortBy` (`relevance`) and `facetQueryResults` (counts per employment type, posted date `ONE`/`THREE`/`SEVEN`, employer type, workplace type, easy apply, willing to sponsor).

## Gotchas

- Use the `/jobs/q-...-jobs` path, not the `/jobs?q=...` query form: robots.txt disallows `/jobs?q*`.
- Hyphens split the path: `q-Front-End+Developer-jobs` was read as "Front" (top hits were "Front Office ..." and "Front Desk ..."). Replace hyphens with spaces first; `q-Front+End+Developer-jobs` returned real front-end roles. `Winston-Salem, NC` as `Winston+Salem%2C+NC` still resolved to "Winston-Salem, NC, USA".
- URL-encode with `quote_plus`: `C#` must be `C%23` (`/jobs/q-C%23+Developer-jobs` worked), commas become `%2C`.
- 25-page clamp: any `page` above 25 silently returns page 25, so compare `meta.currentPage` to the page you asked for. `pageCount` can exceed 25 (46 for "Python Developer"); 500 jobs is the per-search ceiling. Do not split one search into many slices to get around it.
- `pageSize` query param is ignored; always 20 per page. `page=0` returns page 1.
- Unstable order: results are always sorted by `relevance` and the order reshuffles between requests, so consecutive pages repeat some jobs and skip others. Unique jobs over pages 1-5 (100 slots) measured on 2026-10-23: "Python Developer" 99, "Data Engineer" in Chicago 91, "DevOps Engineer" 89, "DevOps Engineer" + `l-Remote` 78. Reloading the same page 2 returned only 8 of the same 20 jobs. Small result sets came back complete (Houston "Python Developer": 132 of 132 over 7 pages). `sortBy=date`, `sort=date`, `sortBy=postedDate` were all ignored (`meta.sortBy` stayed `relevance`). Always de-dup on `guid` and report `len(jobs)` against `totalResults` instead of assuming full coverage.
- The same title can be posted twice under different `guid`s (seen for one recruiter); those are separate postings, keep both.
- `l-Remote` does not filter to remote jobs (top hits were On-Site); use `filters.workplaceTypes=Remote`.
- Counts drift between requests (918 vs 919 total within minutes) because postings update continuously.
- No cookie/consent banner appeared on these pages in the cloud browser (US). No region redirect seen.
- Polite pacing: 2 s between pages was used for the whole run without any 429. On 403, 429 or a page without `jobList`, stop and back off rather than retrying aggressively.

## What does not work / not used

- `/jobs?q=...`, `/jobsearch/`, `/job/...`, `/apply-redirect*`, `/profile/`, `/rss/`, `/feed/`: disallowed in robots.txt, not requested.
- Reading `window.self.__next_f` in the browser after load: it is an empty array (length 0) once hydrated; read the inline `<script>` text instead.
- Unknown keywords return an empty page cleanly: `/jobs/q-zzqqxxnotarealskill-jobs` gave `data: []`, `pageCount: 0`, `totalResults: 0`.
- Applying and saved/recommended jobs were not tested (account features, out of scope for logged-out scraping).
