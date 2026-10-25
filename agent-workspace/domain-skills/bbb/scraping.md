# BBB — Business Search by Category + Location, Ratings, Accreditation and Complaint Counts

Field-tested against www.bbb.org on 2026-10-23 using a Browser Use cloud browser (plus plain HTTP for sitemaps).

robots.txt (fetched 2026-10-23) for `User-agent: *` disallows every URL with a query string (`Disallow: /*?`) except profile pages, so `/search?find_text=...` and `?page=2` pagination are off limits; everything below requests only query-free category pages, profile pages and the published sitemaps.

## Quick summary

- Category + location pages (`/us/<state>/<city>/category/<category>`) are server-rendered and carry the result list in a `window.__PRELOADED_STATE__ = {...};` script tag: name, BBB rating, accreditation flag, street, city, state, postal code, phone, categories, profile URL. 15 rows per page plus `totalResults`.
- Plain HTTP (`curl`/`urllib`) to category and profile pages gets HTTP 403 with a Cloudflare "Just a moment..." page. A real browser loads them with no challenge. Sitemaps (`/sitemap-*.xml`) return 200 over plain HTTP.
- Only page 1 of each listing is robots-allowed. Scale comes from many query-free URLs: one category across many cities, the `/accredited` variant, state-level pages (`/us/<state>/category/<category>`), and the related categories each page lists.
- Business profile pages carry complaint counts (last 3 years, last 12 months), review count, average stars, BBB rating, accreditation and website in the same preloaded state, plus a schema.org `LocalBusiness` JSON-LD block as a cross-check.
- No name search: BBB's own search is under `/search?...`, which robots.txt disallows.

## What works

### 1. Search a category across several cities (browser)

Self-contained. Opens one tab, reads page 1 of the unfiltered and `/accredited` listing per city, dedupes, saves JSON. About 3 s between page loads.

```python
import json, os, re, tempfile, time

def bbb_state():
    # The page deletes window.__PRELOADED_STATE__ after hydrating, so parse the script text instead.
    t = js("(() => { const s = [...document.scripts].find(s => /__PRELOADED_STATE__\\s*=/.test(s.textContent)); return s ? s.textContent : null })()")
    if not t:
        return None
    # Seen both as `__PRELOADED_STATE__ = {...};` and minified `__PRELOADED_STATE__={...}`.
    return json.loads(t.split("=", 1)[1].strip().rstrip(";"))

def bbb_open(url, must_contain, settle=2.0):
    goto_url(url)
    wait_for_load()
    time.sleep(settle)
    href, title = js("location.href"), js("document.title") or ""
    if "Just a moment" in title:
        raise RuntimeError(f"Cloudflare challenge on {url}: stop and back off, do not retry in a loop")
    if "Page not found" in title or must_contain not in href:
        return None
    return href

SLUG = re.compile(r"^[a-z0-9-]+$")

def bbb_category(country, state, city, category, accredited=False):
    """Page 1 of a query-free listing. country 'us' or 'ca'; city=None for the state-level page.
    Returns None for unknown slugs."""
    parts = [country, state] + ([city] if city else []) + ["category", category]
    if not all(SLUG.match(p) for p in parts):
        raise ValueError("slugs must be lowercase-hyphenated, e.g. 'san-francisco', 'plumber'")
    url = "https://www.bbb.org/" + "/".join(parts) + ("/accredited" if accredited else "")
    if not bbb_open(url, "/category/" + category):
        return None
    sr = (bbb_state() or {}).get("searchResult")
    if not sr:
        return None
    loc = (sr.get("location") or {}).get("displayText")
    if city and not loc:
        return None  # unknown city slug: BBB silently serves the state-level list instead
    rows = []
    for r in sr.get("results") or []:
        rows.append({
            "id": r["id"],
            "name": (r.get("businessName") or "").strip(),
            "rating": r.get("rating"),
            "accredited_flag": r.get("bbbMember"),
            "street": r.get("address") or None,
            "city": (r.get("city") or "").strip(),
            "state": r.get("state"),
            "postal_code": r.get("postalcode"),
            "phone": (r.get("phone") or [None])[0],
            "primary_category": r.get("tobText"),
            "categories": [c["name"] for c in r.get("categories") or []],
            "profile_url": "https://www.bbb.org" + r["reportUrl"],
        })
    return {
        "url": url,
        "location": loc,
        "total_results": sr.get("totalResults"),
        "related_categories": [c["url"].rsplit("/", 1)[1] for c in sr.get("relatedCategories") or []],
        "rows": rows,
    }

new_tab("about:blank")
cities = ["san-francisco", "oakland", "san-jose", "berkeley", "daly-city", "not-a-real-city"]
found, pages, last = {}, 0, None
for city in cities:
    for acc in (False, True):
        p = bbb_category("us", "ca", city, "plumber", accredited=acc)
        pages += 1
        time.sleep(3)
        if p is None:
            print(f"{city:15} acc={acc!s:5} -> no listing")
            continue
        last = p
        for row in p["rows"]:
            found.setdefault(row["profile_url"], row)  # one profile can appear twice in a listing
        print(f'{city:15} acc={acc!s:5} total={p["total_results"]:<5} rows={len(p["rows"])} unique_so_far={len(found)}')
print(pages, "pages ->", len(found), "unique businesses")
print(json.dumps(next(iter(found.values()), None), indent=1))
print("related:", last["related_categories"][:5] if last else None)
out = os.path.join(tempfile.gettempdir(), "bbb_plumbers.json")
with open(out, "w") as f:
    json.dump(list(found.values()), f, indent=1)
print("saved", out)
```

Output from the test run (trimmed):

```
san-francisco   acc=False total=4426  rows=15 unique_so_far=15
san-francisco   acc=True  total=640   rows=15 unique_so_far=15
oakland         acc=False total=4588  rows=15 unique_so_far=27
oakland         acc=True  total=667   rows=15 unique_so_far=27
san-jose        acc=False total=4713  rows=15 unique_so_far=42
...
daly-city       acc=True  total=635   rows=15 unique_so_far=61
not-a-real-city acc=False -> no listing
not-a-real-city acc=True  -> no listing
12 pages -> 61 unique businesses
{
 "id": "1116_116593_30865",
 "name": "Atlas Plumbing & Rooter",
 "rating": "A+",
 "accredited_flag": true,
 "street": "3311 Mission St Ste 165",
 "city": "San Francisco",
 "state": "CA",
 "postal_code": "94110-5008",
 "phone": "(415) 845-9199",
 "primary_category": "Plumber",
 "categories": ["Plumber", "Sewer Cleaning", "Water Heater Repair", "Tankless Water Heaters"],
 "profile_url": "https://www.bbb.org/us/ca/san-francisco/profile/plumber/atlas-plumbing-rooter-1116-116593"
}
related: ['plumbing-renovation', 'commercial-plumber', 'water-heater-installation', 'water-heater-repair', 'sewer-cleaning']
```

The `/accredited` pass added no new businesses in any city (same top 15), and neighboring cities overlap (Oakland/Berkeley share rows). To grow coverage, loop over more cities and over `related_categories` instead.

### 2. Profiles: complaint counts, rating, accreditation, website (browser)

Self-contained. Reads the JSON written by block 1 if present and enriches the first rows, then shows a profile with many complaints. About 3 s between profiles.

```python
import json, os, tempfile, time

def bbb_state():
    t = js("(() => { const s = [...document.scripts].find(s => /__PRELOADED_STATE__\\s*=/.test(s.textContent)); return s ? s.textContent : null })()")
    if not t:
        return None
    # Seen both as `__PRELOADED_STATE__ = {...};` and minified `__PRELOADED_STATE__={...}`.
    return json.loads(t.split("=", 1)[1].strip().rstrip(";"))

def bbb_open(url, must_contain, settle=2.0):
    goto_url(url)
    wait_for_load()
    time.sleep(settle)
    href, title = js("location.href"), js("document.title") or ""
    if "Just a moment" in title:
        raise RuntimeError(f"Cloudflare challenge on {url}: stop and back off, do not retry in a loop")
    if "Page not found" in title or must_contain not in href:
        return None
    return href

def bbb_profile(profile_url):
    href = bbb_open(profile_url, "/profile/")
    if not href:
        return None
    bp = (bbb_state() or {}).get("businessProfile")
    if not bp:
        return None
    s = bp.get("reviewsComplaintsSummary") or {}
    loc = bp.get("location") or {}
    ld = {}
    for raw in js("[...document.querySelectorAll('script[type=\"application/ld+json\"]')].map(s => s.textContent)") or []:
        try:
            data = json.loads(raw)
        except ValueError:
            continue
        for x in data if isinstance(data, list) else [data]:
            if x.get("@type") == "LocalBusiness":
                ld = {p["name"]: p["value"] for p in x.get("additionalProperty") or []}
    return {
        "profile_url": href,  # can differ from the listing URL (e.g. a headquarters profile)
        "name": (bp.get("names") or {}).get("primary"),
        "rating": (bp.get("rating") or {}).get("bbbRating"),
        "accredited": (bp.get("accreditationInformation") or {}).get("isAccredited"),
        "complaints_3y": s.get("totalClosedComplaintsPastThreeYears"),
        "complaints_12m": s.get("totalClosedComplaintsPastTwelveMonths"),
        "reviews": s.get("reviewsTotal"),
        "avg_stars": s.get("averageOfReviewStarRatings"),
        "website": (bp.get("urls") or {}).get("primary"),
        "phone": (bp.get("contactInformation") or {}).get("phoneNumber"),
        "address": (loc.get("formattedAddress") or "").lstrip(", "),
        "jsonld_complaints_3y": ld.get("BBB Complaints Filed in Last 3 Years"),
        "jsonld_accreditation": ld.get("BBB Accreditation"),
    }

src = os.path.join(tempfile.gettempdir(), "bbb_plumbers.json")
rows = json.load(open(src)) if os.path.exists(src) else []
for row in rows[:5]:
    prof = bbb_profile(row["profile_url"])
    time.sleep(3)
    if prof is None:
        print("no profile:", row["profile_url"])
        continue
    print(f'{row["name"][:34]:34} {prof["rating"]:3} acc={prof["accredited"]!s:5} '
          f'complaints_3y={prof["complaints_3y"]} 12m={prof["complaints_12m"]} reviews={prof["reviews"]} site={prof["website"]}')

print(json.dumps(bbb_profile("https://www.bbb.org/us/ny/new-york/profile/telephones/verizon-communications-0011-1568"), indent=1))
```

Output from the test run (trimmed):

```
Atlas Plumbing & Rooter            A+  acc=True  complaints_3y=1 12m=0 reviews=1 site=https://www.atlasplumbingandrooter.com
A G Quality Plumbing               A+  acc=True  complaints_3y=0 12m=0 reviews=1 site=https://www.agqualityplumbing.com/
Jimenez Construction               A+  acc=True  complaints_3y=0 12m=0 reviews=1 site=https://jimenezconstructionsf.com/
J. E. Brown Mechanical Co., Inc.   A+  acc=True  complaints_3y=0 12m=0 reviews=0 site=None
Mr. Rooter Plumbing of San Francis A+  acc=True  complaints_3y=0 12m=0 reviews=0 site=https://www.mrrooter.com/san-francisco/
{
 "profile_url": "https://www.bbb.org/us/ny/new-york/profile/telephones/verizon-communications-0011-1568",
 "name": "Verizon Communications",
 "rating": "A+",
 "accredited": false,
 "complaints_3y": 3730,
 "complaints_12m": 871,
 "reviews": 718,
 "avg_stars": 1.1,
 "website": "https://www.verizon.com/",
 "phone": "(800) 779-4116",
 "address": "140 West St, New York, NY 10007-2141",
 "jsonld_complaints_3y": "3730",
 "jsonld_accreditation": "Not BBB Accredited"
}
```

### 3. Discover valid category slugs for a state from the sitemap (plain Python, no browser)

Category slugs are not guessable (`telephone-companies` is a 404, `telephones` works). The state-category sitemaps list them, sorted by state.

```python
import re, time, urllib.request

UA = "bbb-sitemap-reader/1.0 (python-urllib)"

def fetch(url, tries=3):
    delay = 5
    for i in range(tries):
        try:
            req = urllib.request.Request(url, headers={"User-Agent": UA})
            with urllib.request.urlopen(req, timeout=60) as r:
                return r.read().decode("utf-8-sig")
        except Exception as e:
            if i == tries - 1:
                raise RuntimeError(f"{url}: {e}")
            time.sleep(delay)  # 5 s then 10 s, well under 120 s total
            delay *= 2

def state_category_slugs(state):
    idx = fetch("https://www.bbb.org/sitemap-us-state-categories-index.xml")
    slugs, seen_state = [], False
    for sm in re.findall(r"<loc>([^<]+)</loc>", idx):
        locs = re.findall(r"<loc>https://www\.bbb\.org/us/([a-z]{2})/category/([^<]+)</loc>", fetch(sm))
        mine = [c for s, c in locs if s == state]
        slugs += mine
        if mine:
            seen_state = True
        elif seen_state:
            break  # files are sorted by state, so we are past it
        time.sleep(1)
    return slugs

slugs = state_category_slugs("ca")
print(len(slugs), "category slugs for CA")
print([s for s in slugs if "plumb" in s][:10])
```

Output from the test run:

```
2564 category slugs for CA
['wholesale-plumbing-supplies', 'plumbing-fixtures', 'plumbing-and-heating', 'plumber', 'plumbing-plan', 'propane-gas-plumbing', 'plumbing-renovation', 'commercial-plumber', 'plumbing-supplies']
```

## Fields

| Field | Source | Notes |
|---|---|---|
| `id` | listing `results[].id` | `<bbbId>_<businessId>_<n>` |
| `name` | `businessName` | may carry trailing spaces |
| `rating` | `rating` | `A+` ... `F`, `NR` (not rated) |
| `accredited_flag` | `bbbMember` | matched profile `accreditationInformation.isAccredited` for every listing row checked (14 profiles) |
| `street` | `address` | empty string when the business hides its street (both seen) |
| `city`, `state`, `postal_code` | `city`, `state`, `postalcode` | ZIP+4 in the US |
| `phone` | `phone[0]` | business phone; a list in the raw data |
| `categories` | `categories[].name` | BBB "type of business" names |
| `profile_url` | `reportUrl` | relative path, prefix `https://www.bbb.org`; dedupe key (the same profile appeared twice in one listing with different `id`s) |
| `total_results` | `searchResult.totalResults` | size of the whole listing; only the first 15 are robots-allowed |
| `complaints_3y` / `complaints_12m` | profile `reviewsComplaintsSummary.totalClosedComplaintsPastThreeYears` / `...PastTwelveMonths` | JSON-LD `additionalProperty` "BBB Complaints Filed in Last 3 Years" matches |
| `reviews`, `avg_stars` | `reviewsTotal`, `averageOfReviewStarRatings` | 0 when no reviews |
| `website` | profile `urls.primary` | not present in listings |
| `rating` (profile) | `rating.bbbRating` | `NR` comes with `ratingReasonNotRated` text |

## Gotchas

- `window.__PRELOADED_STATE__` is `undefined` by the time you run JS; parse the `<script>` text that assigns it. The same URL was served once as `__PRELOADED_STATE__ = {...};` and later minified as `__PRELOADED_STATE__={...}`, so match with a regex, not a fixed string. The same object holds a `user` section with session cookies and an IP-derived location; read only `searchResult` / `businessProfile`.
- Page 1 is sorted "Best Match". In every listing tried (plumbers, movers, telephones, used car dealers; 6 cities and 1 state page) all 15 page-1 rows had `bbbMember: true` and a rating of B or better. Unaccredited and low-rated businesses sit on pages 2+, which robots.txt disallows. Page 1 is not a representative sample.
- `/accredited` pages report a smaller `totalResults` (San Francisco plumbers: 640 vs 4426) but page 1 was the same 15 businesses as the unfiltered page.
- Listings mix categories: a "plumber" page included rows whose primary category is building contractors or boiler repair.
- Unknown city slug: the URL keeps the fake city but BBB serves the state-level list (`location.displayText` is `None`, heading location "USA"). The helper returns `None`. Unknown category slug: title "Page not found", no state.
- One company can have several BBB profiles under different local BBBs with different accreditation: the New York "telephones" listing links Verizon Communications at `...-0121-411` (accredited, 3,730 complaints in 3 years), while `...-0011-1568` is "Not BBB Accredited" with the same complaint counts. Report the profile URL with every number.
- A listing's `reportUrl` can point to another state (a San Francisco movers row linked to a Massachusetts headquarters profile). Use the final `location.href`.
- Cookie banner ("Accept All Cookies") is shown but does not block the embedded data; no click needed.
- Canada uses the same layout under `/ca/<province>/<city>/category/<category>` (Toronto plumbers: 1,259 results, 15 rows).
- Profile JSON-LD and `contactInformation` can include named owners/contacts and an obfuscated email. Leave them out unless you have a reason and a lawful basis to store them.
- No challenge or rate limiting was hit in about 85 browser page loads at 2 to 3 s spacing. If a "Just a moment..." title appears, the helpers raise; stop and back off.

## What does not work

- Plain HTTP to category or profile pages: HTTP 403, Cloudflare "Just a moment..." page (curl, 2026-10-23). Use the browser; do not change user agents to get around it.
- Name search (`/search?find_text=...&find_loc=...`) and listing pagination (`?page=2` ... `?page=15`, linked from the page's `pagination` nav): disallowed by robots.txt for `User-agent: *` (`Disallow: /*?`). Not used.
- Individual complaint texts: the `/complaints` page server-renders only the summary ("3,730 complaints in the last 3 years. 871 complaints in the last 12 months." for the Verizon profile); `customerComplaints.items` was empty (`numFound: 0`) and the page said "There are no matching results." Counts work, complaint bodies do not this way.
- Finding a business by name via sitemaps is impractical: `sitemap-business-profiles-index.xml` lists 578 files of 10,000 profile URLs each (about 2.3 MB per file).
