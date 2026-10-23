# Upwork — Public Job Posts (title, budget, experience, skills, client country, proposals)

Field-tested against www.upwork.com on 2026-10-23 using a Browser Use cloud browser and `http_get`.

## Quick summary

- Public, logged-out job listings live at `/freelance-jobs/<skill-slug>/` (e.g. `/freelance-jobs/python/`). Each page embeds **30 jobs** in Nuxt's `__NUXT_DATA__` JSON, plus the open-job total for that skill (python: 1,631; web-scraping: 291 on 2026-10-23). The 30 are a recent window sorted oldest-first, not the 30 newest (see Gotchas), and there is no page 2.
- Each job links to a public detail page `/freelance-jobs/apply/<slug>_~<id>/` whose `__NUXT_DATA__` has the fields the listing lacks: **client country** (`buyer.location.country`), **exact proposals count** (`job.clientActivity.totalApplicants`), hourly range, full skill list.
- `http_get` with a truthful client User-Agent returned 200 for the hub, skill pages and detail pages. `http_get`'s bare default UA (`Mozilla/5.0`) and `Python-urllib/3.14` got 403. The browser fallback (read `__NUXT_DATA__` from the loaded tab) also works.
- Scale = many skill pages, not deep pagination. The hub `/freelance-jobs/` lists 482 skill pages.
- `robots.txt` allows `/freelance-jobs/` (except `/freelance-jobs/fiction-writing/`) and disallows `/nx/` (the logged-in search UI), `/api*` and `/*?page=*`. Stay on `/freelance-jobs/`.

Field runs: 2 skill pages (web-scraping, python) -> 60 cards -> 57 unique jobs with detail data in about 3 minutes at 2 s pacing. With retries, each run had exactly one detail miss: a private listing that stays 403 (`detail_error` set, country/proposals `null`).

## What works

### 0. Shared helpers (run before block 1 or 2, in the same script)

Runs as a browser-harness script (`http_get` is pre-imported). No tab needed. `fetch` retries 403/429/5xx and network errors with backoff, honours `Retry-After` (seconds or HTTP-date), (capped at 120 s) and raises `FetchError` instead of returning partial data.

```python
import re, json, time, email.utils, http.client, urllib.error
from datetime import datetime, timezone
from urllib.parse import quote

BASE = "https://www.upwork.com"
HEADERS = {
    # Truthful client UA: 200 on 2026-10-23. http_get's default "Mozilla/5.0" got 403.
    "User-Agent": "browser-harness/1.0 (+https://github.com/browser-use/browser-harness)",
    "Accept-Language": "en-US,en;q=0.9",
}
RETRY_STATUS = {403, 429, 500, 502, 503, 504}   # 403 is often transient here; private listings stay 403
WRAPPERS = ("ShallowReactive", "Reactive", "Ref", "ShallowRef")

class FetchError(Exception):
    def __init__(self, url, status, note=""):
        super().__init__(f"{status} {url} {note}".strip())
        self.url, self.status = url, status

def text(v):
    return v if isinstance(v, str) and v else None

def num(v):
    return v if isinstance(v, (int, float)) and not isinstance(v, bool) else None

def dget(d, *keys):
    for k in keys:
        d = d.get(k) if isinstance(d, dict) else None
    return d

def retry_after_s(value, default):
    """Retry-After is either delay-seconds or an HTTP-date."""
    v = (value or "").strip()
    if v.isdigit():
        return int(v)
    try:
        dt = email.utils.parsedate_to_datetime(v)
    except (TypeError, ValueError):
        return default
    if dt.tzinfo is None:
        dt = dt.replace(tzinfo=timezone.utc)
    return max(0.0, (dt - datetime.now(timezone.utc)).total_seconds())

def fetch(url, tries=3):
    """http_get with backoff (5 s, 10 s). Returns HTML or raises FetchError."""
    for attempt in range(tries):
        wait = 5 * 2 ** attempt
        try:
            return http_get(url, headers=HEADERS)
        except urllib.error.HTTPError as e:
            status = e.code
            if status not in RETRY_STATUS:
                raise FetchError(url, status) from e
            wait = retry_after_s(e.headers.get("Retry-After"), wait)
        except (urllib.error.URLError, TimeoutError, ConnectionError, http.client.HTTPException) as e:
            status = type(e).__name__
        if attempt == tries - 1:
            raise FetchError(url, status, f"after {tries} tries")
        print(f"  {status} on {url}; retry in {wait:.0f} s")
        time.sleep(min(wait, 120))

def nuxt_payload(html):
    """Decode Nuxt 3 __NUXT_DATA__ (devalue format: one flat array, dict/list values are indexes into it)."""
    m = re.search(r'<script[^>]*id="__NUXT_DATA__"[^>]*>(.*?)</script>', html or "", re.S)
    if not m:
        return None
    flat, memo = json.loads(m.group(1)), {}
    def ref(i):
        if not isinstance(i, int) or isinstance(i, bool) or not 0 <= i < len(flat):   # -1 undefined, -2 null
            return None
        if i not in memo:
            v = flat[i]
            if isinstance(v, list):
                memo[i] = ref(v[1]) if len(v) == 2 and v[0] in WRAPPERS else [ref(x) for x in v]
            elif isinstance(v, dict):
                memo[i] = {k: ref(x) for k, x in v.items()}
            else:
                memo[i] = v
        return memo[i]
    return ref(0)
```

### 1. Scrape 2+ skill pages and enrich every job from its detail page (`http_get`)

Run block 0 first in the same script.

```python
TIER = {1: "Entry level", 2: "Intermediate", 3: "Expert"}   # 0 = not specified
JOB_TYPE = {1: "Fixed-price", 2: "Hourly"}

def skill_page(slug):
    """One /freelance-jobs/<slug>/ page: its 30 jobs + the category's open-job total. Raises on failure."""
    url = f"{BASE}/freelance-jobs/{quote(slug, safe='')}/"
    for v in (dget(nuxt_payload(fetch(url)), "data") or {}).values():
        if isinstance(v, dict) and isinstance(v.get("jobs"), list) and "totalJobs" in v:
            return v["jobs"], num(v["totalJobs"])
    raise FetchError(url, 200, "no job list in __NUXT_DATA__")

def job_detail(path):
    """Returns (jobDetails dict, error string or None). A miss is explicit, never silent."""
    try:
        html = fetch(BASE + quote(path, safe="/%"))
    except FetchError as e:
        return {}, f"HTTP {e.status}"
    jd = dget(nuxt_payload(html), "vuex", "jobDetails")
    return (jd, None) if isinstance(dget(jd, "job"), dict) else ({}, "no jobDetails in page")

def iso_utc(v):
    s = text(v)   # detail uses "...Z"; the card fallback uses "+0000"
    return s[:-5] + "Z" if s and s.endswith("+0000") else s

def to_record(card, jd, err):
    job, buyer = dget(jd, "job") or {}, dget(jd, "buyer") or {}
    jtype = num(job.get("type")) or num(card.get("type"))
    tier = num(job.get("contractorTier"))
    tier = num(card.get("contractorTier")) if tier is None else tier
    fixed = num(dget(job, "budget", "amount"))
    fixed = num(dget(card, "amount", "amount")) if fixed is None else fixed
    skills = [dget(s, "prefLabel") for s in card.get("ontologySkills") or []]
    skills += [dget(s, "name") for s in dget(jd, "sands", "additionalSkills") or []]
    return {
        "id": text(card.get("ciphertext")),
        "title": text(job.get("title")) or text(card.get("title")),
        "posted_on": iso_utc(job.get("postedOn")) or iso_utc(card.get("createdOn")),
        "posted_ago": text(card.get("ago")),
        "job_type": JOB_TYPE.get(jtype),
        "fixed_budget": fixed if jtype == 1 else None,
        "hourly_min": num(dget(job, "extendedBudgetInfo", "hourlyBudgetMin")) if jtype == 2 else None,
        "hourly_max": num(dget(job, "extendedBudgetInfo", "hourlyBudgetMax")) if jtype == 2 else None,
        "experience_level": TIER.get(tier),
        "skills": list(dict.fromkeys(s for s in skills if text(s))),
        "client_country": text(dget(buyer, "location", "country")),
        "proposals": num(dget(job, "clientActivity", "totalApplicants")),
        "workload": text(job.get("workload")) or text(card.get("engagement")),
        "duration": text(job.get("durationLabel")) or text(card.get("durationLabel")),
        "url": BASE + card["url"],
        "detail_error": err,
    }

SKILLS = ["web-scraping", "python"]      # each slug = one listing page of 30 jobs
rows, seen = [], set()
for slug in SKILLS:
    cards, total = skill_page(slug)      # raises if the listing cannot be read
    print(f"{slug}: {len(cards)} cards of {total} open jobs (no page 2 exists; the rest are not reachable logged out)")
    time.sleep(2)
    for card in cards:
        if not isinstance(card, dict) or not text(card.get("url")):
            print("  card without url, skipped:", card)
            continue
        key = text(card.get("ciphertext")) or card["url"]   # never drop a card just because its id is missing
        if key in seen:
            continue
        seen.add(key)
        jd, err = job_detail(card["url"])
        if err:
            print("  detail failed:", err, card["url"])
        rows.append(to_record(card, jd, err))
        time.sleep(2)                    # polite pacing: ~1 request per 2-3 s

with open("upwork_jobs.jsonl", "w") as f:
    for r in rows:
        f.write(json.dumps(r) + "\n")
print(len(rows), "jobs saved;", sum(r["detail_error"] is not None for r in rows), "without detail data")
print(json.dumps(rows[:2], indent=1))
print("missing country:", sum(r["client_country"] is None for r in rows),
      "missing proposals:", sum(r["proposals"] is None for r in rows))
```

Observed output (trimmed; which job gets a transient 403 varies by run):

```text
web-scraping: 30 cards of 291 open jobs (no page 2 exists; the rest are not reachable logged out)
python: 30 cards of 1631 open jobs (no page 2 exists; the rest are not reachable logged out)
  403 on https://www.upwork.com/freelance-jobs/apply/Senior-Backend-GCP-Data-Engineer_~022113259734270443656/; retry in 5 s
  403 on https://www.upwork.com/freelance-jobs/apply/Port-Viva-com-payment-module-Odoo-and-add-ISV-key_~022113260749740632003/; retry in 5 s
  403 on https://www.upwork.com/freelance-jobs/apply/Port-Viva-com-payment-module-Odoo-and-add-ISV-key_~022113260749740632003/; retry in 10 s
  detail failed: HTTP 403 /freelance-jobs/apply/Port-Viva-com-payment-module-Odoo-and-add-ISV-key_~022113260749740632003/
57 jobs saved; 1 without detail data
[
 {
  "id": "~022112614550818987209",
  "title": "E-commerce Product Monitoring System",
  "posted_on": "2026-10-20T18:38:54.416Z",
  "posted_ago": "3 days ago",
  "job_type": "Fixed-price",
  "fixed_budget": 100,
  "hourly_min": null,
  "hourly_max": null,
  "experience_level": "Intermediate",
  "skills": ["Selenium", "Data Scraping", "HTML", "Python", "Web Scraping", "Web Scraping Framework", "Playwright", "Data Extraction", "AI Agents", "pandas"],
  "client_country": "IND",
  "proposals": 26,
  "workload": null,
  "duration": "1 to 3 months",
  "url": "https://www.upwork.com/freelance-jobs/apply/commerce-Product-Monitoring-System_~022112614550818987209/",
  "detail_error": null
 },
 ...
]
missing country: 1 missing proposals: 1
```

Distribution (same on two runs today): 31 fixed-price / 26 hourly; Intermediate 34, Expert 19, Entry level 3, unspecified 1; proposals ranged 0 to 660.

### 2. Turn a keyword into skill slugs (the robots-allowed "search")

There is no public keyword search with pagination under `/freelance-jobs/`. The hub page `/freelance-jobs/` embeds the 482 skill pages it links to (`id`, `url`, `title`). Match your keyword against them, then feed the slugs to block 1. Run block 0 first in the same script.

```python
ROBOTS_DISALLOWED = {"fiction-writing"}   # robots.txt: Disallow: /freelance-jobs/fiction-writing/

skill_pages = None
for v in (dget(nuxt_payload(fetch(f"{BASE}/freelance-jobs/")), "data") or {}).values():
    if isinstance(v, dict) and "categories" in v and isinstance(v.get("jobs"), list):
        skill_pages = [p for p in v["jobs"] if isinstance(p, dict) and text(p.get("id"))]
if not skill_pages:
    raise FetchError(f"{BASE}/freelance-jobs/", 200, "skill list not found in __NUXT_DATA__")
print(len(skill_pages), "skill listing pages")

def find_skill_slugs(keyword):
    kw = keyword.lower().strip()
    return [p["id"] for p in skill_pages
            if p["id"] not in ROBOTS_DISALLOWED
            and (kw in p["id"].replace("-", " ") or kw in (text(p.get("title")) or "").lower())]

for kw in ["scraping", "react", "data entry", "python"]:
    print(kw, "->", find_skill_slugs(kw))
```

Observed output:

```text
482 skill listing pages
scraping -> ['data-scraping', 'web-scraping']
react -> ['react-js', 'react-native']
data entry -> ['data-entry']
python -> ['python-script', 'python']
```

Opening `https://www.upwork.com/o/jobs/browse/?q=<keyword>` (URL-encode the keyword with `quote(kw, safe='')`) in the browser redirected to the matching skill page for known skills (`shopify` -> `/freelance-jobs/shopify/`, `data entry` -> `/freelance-jobs/data-entry/`, `react native` -> `/freelance-jobs/react-native/`); a nonsense query (`zzqxv`) showed a Cloudflare "Worker threw exception" page instead.

### 3. Browser fallback (same data, read from the tab)

Use this when `http_get` returns 403 from your network. It reads the same `__NUXT_DATA__` from the rendered page. `open_page` confirms the tab's path matches the requested page and its document is complete before parsing, and retries a browser timeout once. Works on a background tab; it never calls `activate_tab`.

```python
import json, time
from urllib.parse import urlsplit

WRAPPERS = ("ShallowReactive", "Reactive", "Ref", "ShallowRef")

def is_timeout(e):   # cdp() raises TimeoutError; js() wraps it as RuntimeError("Runtime.evaluate timed out ...")
    return isinstance(e, TimeoutError) or "timed out" in str(e)

def open_page(url, first=False, tries=2):
    """Navigate, then wait until the tab shows THIS url's document, complete, with __NUXT_DATA__."""
    want = urlsplit(url).path.rstrip("/")
    for attempt in range(tries):
        try:
            new_tab(url) if first and attempt == 0 else goto_url(url)
            deadline = time.time() + 20
            while time.time() < deadline:
                state = js("JSON.stringify([location.pathname, document.readyState, !!document.getElementById('__NUXT_DATA__')])")
                path, ready, has_data = json.loads(state) if isinstance(state, str) else ("", "", False)
                if path.rstrip("/") == want and ready == "complete" and has_data:
                    return
                time.sleep(0.5)
        except (TimeoutError, RuntimeError) as e:
            if not is_timeout(e):
                raise
            print("  browser timeout, retrying:", e)
    raise RuntimeError(f"{url} did not load; tab is at {js('location.href')}")

def page_payload():
    """Read and decode __NUXT_DATA__ from the page currently loaded in the tab."""
    raw = js("(document.getElementById('__NUXT_DATA__') || {}).textContent || ''")
    if not isinstance(raw, str) or not raw:
        return None
    flat, memo = json.loads(raw), {}
    def ref(i):
        if not isinstance(i, int) or isinstance(i, bool) or not 0 <= i < len(flat):
            return None
        if i not in memo:
            v = flat[i]
            if isinstance(v, list):
                memo[i] = ref(v[1]) if len(v) == 2 and v[0] in WRAPPERS else [ref(x) for x in v]
            elif isinstance(v, dict):
                memo[i] = {k: ref(x) for k, x in v.items()}
            else:
                memo[i] = v
        return memo[i]
    return ref(0)

open_page("https://www.upwork.com/freelance-jobs/data-entry/", first=True)
cards, total = None, None
for v in ((page_payload() or {}).get("data") or {}).values():
    if isinstance(v, dict) and isinstance(v.get("jobs"), list) and "totalJobs" in v:
        cards, total = v["jobs"], v["totalJobs"]
if cards is None:
    raise RuntimeError("no job list in __NUXT_DATA__ on the skill page")
print(len(cards), "cards of", total, "open jobs")

for card in cards[:3]:
    try:
        open_page("https://www.upwork.com" + card["url"])
    except RuntimeError as e:
        print("  detail failed:", e)
        continue
    jd = ((page_payload() or {}).get("vuex") or {}).get("jobDetails") or {}
    job, buyer = jd.get("job") or {}, jd.get("buyer") or {}
    print(str(card.get("title"))[:60], "|", (buyer.get("location") or {}).get("country"),
          "|", (job.get("clientActivity") or {}).get("totalApplicants"), "proposals")
    time.sleep(2)
```

Observed output:

```text
30 cards of 584 open jobs
Data Entry Specialist | United States | 137 proposals
Data entry assistant needed for general research project | United States | 156 proposals
Social Media Handle Verification Specialist | USA | 126 proposals
```

## Fields

Block 1 returns one dict per job with exactly these keys. Listing card = item in the skill page's `data[...].jobs`. Detail = `vuex.jobDetails` on the job page.

| Output field | Source | Notes |
|---|---|---|
| `id` | card `ciphertext` | e.g. `~022112614550818987209`; used to dedupe across skill pages (falls back to `url` if missing) |
| `title` | detail `job.title` / card `title` | |
| `posted_on` | detail `job.postedOn` (fallback card `createdOn`) | ISO UTC; the card's `+0000` suffix is rewritten to `Z`. Card has `renewedOn` (`null` unless renewed) |
| `posted_ago` | card `ago` | e.g. `"1 day ago"`, `"9 hours ago"` |
| `job_type` | `type` | `1` = Fixed-price, `2` = Hourly |
| `fixed_budget` | detail `job.budget.amount` / card `amount.amount` | USD, fixed-price only (`null` for hourly; card `amount.amount` is `0` there) |
| `hourly_min`, `hourly_max` | detail `job.extendedBudgetInfo.hourlyBudgetMin/Max` | Hourly only. 7 of 26 hourly jobs had no range (`null`). Not read from the card |
| `experience_level` | `contractorTier` | `1` Entry level, `2` Intermediate, `3` Expert, `0` not specified -> `null` (labels checked against the rendered page) |
| `skills` | card `ontologySkills[].prefLabel` + detail `sands.additionalSkills[].name` | Detail adds skills the page hides behind "+ N more" |
| `client_country` | detail `buyer.location.country` | Not on listing cards. Mixed formats, see Gotchas |
| `proposals` | detail `job.clientActivity.totalApplicants` | Exact integer. The page shows a bucket such as "20 to 50" |
| `workload` | detail `job.workload` / card `engagement` | `null` on fixed-price jobs |
| `duration` | `durationLabel` | e.g. `"1 to 3 months"` |
| `url` | `https://www.upwork.com` + card `url` | `/freelance-jobs/apply/<slug>_~<ciphertext>/` |
| `detail_error` | | `null` when the detail page was read; else why not (`"HTTP 403"`, `"no jobDetails in page"`). Country, proposals and hourly range are `null` then |

Also present but not returned: detail `job.clientActivity` has `totalHired`, `totalInvitedToInterview`, `invitationsSent`, `unansweredInvites`, `lastBuyerActivity`; `buyer.stats` has `totalCharges.amount` (client spend), `totalAssignments`, `score`; plus `job.category.name`, `job.description`. Card `maxAmount.amount` is the max hourly rate (`-1` when unset).

## Gotchas

- **30 jobs per skill page, oldest-first, no page 2.** Cards come in ascending `createdOn` order (one swap per page seen). The 27 python jobs not also on web-scraping were all posted 2026-10-22 04:58-15:06 UTC although the category has 1,631 open jobs; web-scraping (291) spanned 2026-10-20 18:38 to 2026-10-23 09:03. So a big category is not shown as its 30 newest jobs. `/freelance-jobs/python/2/` returned 404 and `?page=2` returned the same 30 ids (and `?page=` is robots-disallowed). To get more jobs, scrape more skill slugs and dedupe on `id` (web-scraping and python shared 3 jobs).
- **"Load more jobs" only reveals what is already embedded.** The rendered python page showed 10 cards; one click on "Load more jobs" showed 20 and the button disappeared. No new job request was made and all shown ids were among the 30 in `__NUXT_DATA__`. The data-entry page rendered all 30 with no button. Read the payload, not the DOM.
- **Client country formats are mixed in the source**, and the page renders them as-is: full names (`"United States"`, `"India"`) and 3-letter codes (`"USA"`, `"IND"`, `"CAN"`, `"GBR"`) appeared in the same run. Normalize if you group by country.
- **JSON-LD is misleading for country.** The detail page's `JobPosting` JSON-LD had `jobLocation.address.addressCountry: "US"` for a client located in India. Use `buyer.location.country`.
- **403s on detail pages.** About 1 in 60 public job pages returned a transient 403 to `http_get` (it loaded fine in the browser and a later retry got it). One job stayed 403 in every run and the browser showed "This job is a private listing." Block 1 retries twice, then keeps the card-level fields and sets `detail_error`.
- **User-Agent and client matter.** With the same headers, `http_get`'s default `Mozilla/5.0` and `Python-urllib/3.14` got 403 while a truthful `browser-harness/1.0 (+url)` UA got 200. Plain `curl` with a Chrome UA got a Cloudflare "Sorry, you have been blocked" 403, and macOS system `python3` 3.9.6 got 403 on the hub where browser-harness's Python 3.14 got 200. If `http_get` is blocked from your setup, use block 3.
- Blocks were tested with `BROWSER_USE_API_KEY` unset, so `http_get` used local urllib. With the key set, `http_get` routes through the fetch-use proxy, which raises different errors than `urllib.error.HTTPError`.
- **No consent banner or login wall** appeared on `/freelance-jobs/` pages in the cloud browser.
- `js()` calls time out after 5 s, so do not run slow multi-request `fetch()` loops inside the page; navigate instead (block 3).
- Pages are large (~1.0-1.4 MB HTML each). Keep 2-3 s between requests; the 60-request runs at that pace saw no 429.

## What does not work

- **`/nx/search/jobs/?q=...` (the real search UI)**: disallowed by `robots.txt`. A single logged-out visit rendered only 10 cards (`paging.total: 12`) with `client.location.country: null` on every card, so it is also not useful logged out.
- **`/sitemap.xml`**: 403 to `http_get`.
- **`?page=N` on skill pages**: ignored; also disallowed by `robots.txt` (`/*?page=*`).
- **Getting more than 30 jobs from one skill page**: not possible logged out. There is no page 2, and "Load more jobs" does not fetch beyond the 30 embedded jobs.
- **Freelancer profiles**: out of scope here (`/freelancers/public/api/` and `/search/profiles/*` are disallowed by `robots.txt`).
