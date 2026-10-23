# Redfin — For-sale listings by city or ZIP, plus listing details

Field-tested against www.redfin.com on 2026-10-23 using a Browser Use cloud browser and plain-Python HTTP (urllib).

## Quick summary

- Redfin's own map/list UI is fed by a same-origin JSON endpoint, `/stingray/api/gis`. Called from inside a Redfin tab with `fetch()`, it returned every for-sale home in a region: address, price, beds, baths, sqft, status, lat/long and URL, plus remarks, key facts, tags, HOA, lot size and year built.
- Pagination is `start=<offset>` with `num_homes=<page size>`. `page_number` is ignored. The city of Austin, TX returned 5,795 unique homes in 17 calls of 350 (about 61 s with a 2 s pause).
- City / ZIP / neighborhood / county text is turned into a `region_id` + `region_type` with `/stingray/do/location-autocomplete`, also called from the tab.
- Listing detail pages carry a `["Product", "RealEstateListing"]` JSON-LD block (price, beds, baths, sqft, year built, geo, photos, description). Three more same-origin endpoints add status, lot size, full/half baths and every MLS fact group.
- Plain HTTP (no browser): `/stingray/api/gis` answered 200 to urllib, so bulk search works without a browser once you know the region id. Autocomplete, detail pages and the detail APIs did not (403 / 202 empty), so use the browser for those.

## What works

### 1. Search a city or ZIP and collect every listing (browser)

Self-contained. Change `QUERY`. Writes JSON and CSV to the system temp dir.

```python
import json, time, csv, os, tempfile, urllib.parse

QUERY = "Austin, TX"        # a city, a ZIP ("78704"), a neighborhood or a county
PAGE_SIZE = 350             # the page size Redfin's own map uses
MAX_PAGES = 30              # safety cap; Austin needed 17 pages
PAUSE = 2.0                 # seconds between API calls

new_tab("https://www.redfin.com/")
wait_for_load(30)

def rf(path, timeout=60):
    """GET a same-origin /stingray endpoint from inside the Redfin tab. Returns parsed JSON or None.
    The fetch runs in the page and is polled, because a single js() call gives up after about 5 s."""
    k = json.dumps("__rf_%d" % time.time_ns())
    js("window[%s] = null; fetch(%s).then(async r => { window[%s] = JSON.stringify({status: r.status, text: await r.text()}); })"
       ".catch(e => { window[%s] = JSON.stringify({status: 0, text: String(e)}); }); 1" % (k, json.dumps(path), k, k))
    raw, deadline = None, time.time() + timeout
    while not raw and time.time() < deadline:
        time.sleep(0.5)
        raw = js("window[%s]" % k)
    js("delete window[%s]; 1" % k)
    r = json.loads(raw) if raw else {"status": "timeout", "text": ""}
    if r["status"] != 200 or not r["text"].startswith("{}&&"):
        print("stingray failed", r["status"], path[:90], r["text"][:120])
        return None
    return json.loads(r["text"][4:])      # strip the "{}&&" prefix

# autocomplete row type -> gis region_type
REGION_TYPE = {"2": 6, "4": 2, "6": 1, "5": 5}   # city, zip, neighborhood, county

def resolve_region(query):
    d = rf("/stingray/do/location-autocomplete?" + urllib.parse.urlencode({"location": query, "v": 2}))
    for sec in (d or {}).get("payload", {}).get("sections", []):
        for row in sec.get("rows", []):
            if row.get("type") in REGION_TYPE:
                return {"region_id": int(row["id"].split("_")[1]), "region_type": REGION_TYPE[row["type"]],
                        "name": row.get("name"), "sub": row.get("subName"), "url": row.get("url")}
    return None

def v(x):
    return x.get("value") if isinstance(x, dict) else x

def flatten(h):
    ll = v(h.get("latLong")) or {}
    return {
        "property_id": h.get("propertyId"), "listing_id": h.get("listingId"), "mls_id": v(h.get("mlsId")),
        "address": v(h.get("streetLine")), "city": h.get("city"), "state": h.get("state"), "zip": h.get("zip"),
        "price": v(h.get("price")), "beds": h.get("beds"), "baths": h.get("baths"), "sqft": v(h.get("sqFt")),
        "lot_sqft": v(h.get("lotSize")), "year_built": v(h.get("yearBuilt")), "price_per_sqft": v(h.get("pricePerSqFt")),
        "hoa_month": v(h.get("hoa")), "days_on_market": v(h.get("dom")), "status": h.get("mlsStatus"),
        "neighborhood": v(h.get("location")), "lat": ll.get("latitude"), "lon": ll.get("longitude"),
        "url": "https://www.redfin.com" + h["url"] if h.get("url") else None,
    }

region = resolve_region(QUERY)
if not region:
    raise SystemExit(f"No Redfin region found for {QUERY!r}")
print("region", region)

q = {"al": 1, "ord": "redfin-recommended-asc", "sf": "1,2,3,5,6,7", "status": 9, "uipt": "1,2,3,4,5,6,7,8", "v": 8,
     "region_id": region["region_id"], "region_type": region["region_type"], "num_homes": PAGE_SIZE}
agg = rf("/stingray/api/gis-aggregates?" + urllib.parse.urlencode(q)) or {}
if agg.get("payload"):
    print("site count:", agg["payload"].get("numProperties"), "+", agg["payload"].get("numEarlyAccess"), "early access")
else:
    print("no aggregate count:", agg.get("resultCode"), agg.get("errorMessage"))

rows, seen = [], set()
for page in range(MAX_PAGES):
    time.sleep(PAUSE)
    d = rf("/stingray/api/gis?" + urllib.parse.urlencode(dict(q, start=page * PAGE_SIZE)))
    homes = (d or {}).get("payload", {}).get("homes", [])
    for h in homes:
        if h.get("propertyId") not in seen:
            seen.add(h.get("propertyId"))
            rows.append(flatten(h))
    print(f"page {page + 1}: {len(homes)} homes, {len(rows)} unique so far")
    if len(homes) < PAGE_SIZE:
        break

out = os.path.join(tempfile.gettempdir(), "redfin_listings")
with open(out + ".json", "w") as f:
    json.dump(rows, f)
if rows:
    with open(out + ".csv", "w", newline="") as f:
        w = csv.DictWriter(f, fieldnames=list(rows[0]))
        w.writeheader()
        w.writerows(rows)
print("saved", len(rows), "rows to", out + ".json", "and .csv")
print(json.dumps(rows[:2], indent=1))
```

Observed output (trimmed):

```text
region {'region_id': 30818, 'region_type': 6, 'name': 'Austin', 'sub': 'Austin, TX, USA', 'url': '/city/30818/TX/Austin'}
site count: 5793 + 120 early access
page 1: 350 homes, 350 unique so far
...
page 16: 350 homes, 5600 unique so far
page 17: 195 homes, 5795 unique so far
saved 5795 rows to /tmp/.../redfin_listings.json and .csv
[
 {"property_id": 32017342, "listing_id": 224635318, "mls_id": "8807705",
  "address": "3305 Tavistock Dr", "city": "Austin", "state": "TX", "zip": "78748",
  "price": 599900, "beds": 3, "baths": 2.5, "sqft": 2190, "lot_sqft": 6638, "year_built": 2007,
  "price_per_sqft": 274, "hoa_month": 48, "days_on_market": 1, "status": "Active",
  "neighborhood": "Bauerle Ranch", "lat": 30.1699391, "lon": -97.8444657,
  "url": "https://www.redfin.com/TX/Austin/3305-Tavistock-Dr-78748/home/32017342"},
 {"property_id": 30979888, "address": "1601 Faro Dr #1303", "zip": "78741", "price": 249900,
  "beds": 2, "baths": 1.0, "sqft": 900, "status": "Coming Soon", "lat": 30.233482, "lon": -97.711697, ...}
]
```

With `QUERY = "78704"` autocomplete returned row id `4_34098`, so the ZIP searches as `region_id=34098, region_type=2` (the ZIP itself is not the region id). That ZIP returned 331 homes (the page header said "321 homes + 10 early access"). `"Travis County, TX"` resolved to `/county/2866/TX/Travis-County` (`region_id=2866, region_type=5`).

Null counts across the 5,795 Austin rows: address 18 (undisclosed-address listings), price 17, beds 17, baths 346, sqft 418, lot_sqft 908, year_built 922, hoa_month 2,876, days_on_market 120 (all Coming Soon). Status values: Active 5,671, Coming Soon 120, Price Change 2, New 1, Back on Market 1. lat/lon and url were never null.

### 2. Key facts from a listing detail page (browser)

Reads the first URL saved by block 1; pass any `https://www.redfin.com/.../home/<id>` URL to `listing_details()` instead.

```python
import json, os, re, time, tempfile, urllib.parse

# Take a listing URL from the search output (block above); any redfin.com/.../home/<id> URL works.
saved = os.path.join(tempfile.gettempdir(), "redfin_listings.json")
listings = json.load(open(saved)) if os.path.exists(saved) else []
URL = listings[0]["url"] if listings else None
if not URL:
    raise SystemExit("Run the search block first, or set URL to a redfin.com listing URL")

if not page_info()["url"].startswith("https://www.redfin.com"):
    new_tab("https://www.redfin.com/")
    wait_for_load(30)

def rf(path, timeout=60):
    """GET a same-origin /stingray endpoint from inside the Redfin tab. Returns parsed JSON or None.
    The fetch runs in the page and is polled, because a single js() call gives up after about 5 s."""
    k = json.dumps("__rf_%d" % time.time_ns())
    js("window[%s] = null; fetch(%s).then(async r => { window[%s] = JSON.stringify({status: r.status, text: await r.text()}); })"
       ".catch(e => { window[%s] = JSON.stringify({status: 0, text: String(e)}); }); 1" % (k, json.dumps(path), k, k))
    raw, deadline = None, time.time() + timeout
    while not raw and time.time() < deadline:
        time.sleep(0.5)
        raw = js("window[%s]" % k)
    js("delete window[%s]; 1" % k)
    r = json.loads(raw) if raw else {"status": "timeout", "text": ""}
    if r["status"] != 200 or not r["text"].startswith("{}&&"):
        print("stingray failed", r["status"], path[:90], r["text"][:120])
        return None
    return json.loads(r["text"][4:])      # strip the "{}&&" prefix

def strip_tags(s):
    return re.sub(r"<[^>]+>", "", s or "").strip()

def listing_details(url):
    goto_url(url)
    wait_for_load(30)
    time.sleep(2)
    if "/home/" not in page_info()["url"]:
        return None                      # listing gone or redirected to a search page
    # 1) JSON-LD block typed ["Product", "RealEstateListing"]
    ld = None
    for t in js("[...document.querySelectorAll('script[type=\"application/ld+json\"]')].map(s => s.textContent)") or []:
        try:
            d = json.loads(t)
        except ValueError:
            continue
        if isinstance(d, dict) and "RealEstateListing" in (d.get("@type") or []):
            ld = d
    me = (ld or {}).get("mainEntity", {})
    offer = (ld or {}).get("offers", {})
    facts = {
        "url": url,
        "address": me.get("address", {}),
        "price": offer.get("price"),
        "availability": (offer.get("availability") or "").rsplit("/", 1)[-1],
        "date_posted": (ld or {}).get("datePosted"),
        "home_type": me.get("accommodationCategory"),
        "beds": me.get("numberOfBedrooms"),
        "baths": me.get("numberOfBathroomsTotal"),
        "sqft": (me.get("floorSize") or {}).get("value"),
        "year_built": me.get("yearBuilt"),
        "lat": (me.get("geo") or {}).get("latitude"),
        "lon": (me.get("geo") or {}).get("longitude"),
        "amenities": [a.get("name") for a in me.get("amenityFeature", [])],
        "photos": [i.get("url") for i in me.get("image", [])],
        "description": (ld or {}).get("description", "").strip(),
    }
    # 2) ids for the detail APIs, from the page path
    path = urllib.parse.urlparse(url).path
    info = rf("/stingray/api/home/details/initialInfo?" + urllib.parse.urlencode({"path": path}))
    p = (info or {}).get("payload", {})
    pid, lid = p.get("propertyId"), p.get("listingId")
    if pid:
        q = urllib.parse.urlencode({"propertyId": pid, "listingId": lid or "", "accessLevel": 1})
        time.sleep(1)
        atf = (rf("/stingray/api/home/details/aboveTheFold?" + q) or {}).get("payload", {}).get("addressSectionInfo", {})
        facts.update({
            "property_id": pid, "listing_id": lid, "mls_id": p.get("mlsId"),
            "status": (atf.get("status") or {}).get("displayValue"),
            "lot_sqft": atf.get("lotSize"),
            "price_per_sqft": atf.get("pricePerSqFt"),
            "full_baths": atf.get("numFullBaths"), "half_baths": atf.get("numPartialBaths"),
        })
        time.sleep(1)
        btf = (rf("/stingray/api/home/details/belowTheFold?" + q) or {}).get("payload", {})
        groups = {}
        for sg in (btf.get("amenitiesInfo") or {}).get("superGroups", []):
            for g in sg.get("amenityGroups", []):
                for e in g.get("amenityEntries", []):
                    vals = ", ".join(strip_tags(x) for x in e.get("amenityValues", []))
                    groups.setdefault(g.get("groupTitle") or "untitled", []).append(f'{e.get("amenityName") or e.get("referenceName")}: {vals}')
        facts["mls_details"] = groups
    return facts

d = listing_details(URL)
if d is None:
    print("listing page not available:", URL)
else:
    out = dict(d, description=d["description"][:120] + "...", photos=f'{len(d["photos"])} urls, first: {d["photos"][0] if d["photos"] else None}',
               mls_details={k: v[:3] for k, v in list(d["mls_details"].items())[:6]} if d.get("mls_details") else None)
    print(json.dumps(out, indent=1))
```

Observed output (trimmed):

```text
{
 "url": "https://www.redfin.com/TX/Austin/3305-Tavistock-Dr-78748/home/32017342",
 "address": {"@type": "PostalAddress", "streetAddress": "3305 Tavistock Dr", "addressLocality": "Austin",
             "addressRegion": "TX", "postalCode": "78748", "addressCountry": "US"},
 "price": 599900, "availability": "InStock", "date_posted": "2026-10-23T14:21:33.012Z",
 "home_type": "Single Family Residential", "beds": 3, "baths": 2.5, "sqft": 2190, "year_built": 2007,
 "lat": 30.1699391, "lon": -97.8444657,
 "amenities": ["Heating & cooling: Has A/C", "Parking: 2 car garage"],
 "photos": "36 urls, first: https://ssl.cdn-redfin.com/photo/92/bigphoto/705/8807705_0.jpg",
 "property_id": 32017342, "listing_id": 224635318, "mls_id": "8807705",
 "status": "Active", "lot_sqft": 6638, "price_per_sqft": 274, "full_baths": 2, "half_baths": 1,
 "mls_details": {
  "Garage/Carport Information": ["GARAGE_YN: Has Garage", "# of Garage Spaces: 2"],
  "Parking": ["Features: Driveway, Garage", "# of Parking (Total): 4", "# of Covered Spaces: 2"],
  "Bathroom Information": ["# of Full Bathrooms: 2", "# of Half Bathrooms: 1"],
  ...
 }
}
```

Looping `listing_details()` over 8 Austin listings (condos, houses, one Coming Soon) with a 3 s pause took 8-11 s per listing and never got blocked. MLS groups seen: Parking, Bedroom/Bathroom Information, Room N Information, Laundry, Fireplace, Equipment, Interior/Exterior Features, Building Information, Green Features, Pool and Spa, Lot Information, Property Information, Assessments and Fees, Tax Information, Utility Information, Heating & Cooling, HOA Information, Community Information, Virtual Tour, plus one untitled group holding the public-record facts (Beds, Baths, Finished/Unfinished/Total Sq. Ft., Stories, Lot Size, Style, Year Built, Year Renovated, County, APN).

### 3. Bulk search without a browser (plain Python)

`/stingray/api/gis` answered plain urllib requests with 200 JSON (both with a full Chrome UA and with a bare `Mozilla/5.0`). Get `region_id` / `region_type` once from block 1, then page with this.

```python
import gzip, json, time, urllib.error, urllib.parse, urllib.request

# region_id / region_type come from the browser block's resolve_region() output.
# 34098 / 2 is ZIP 78704 (Austin, TX); 30818 / 6 is the city of Austin.
REGION_ID, REGION_TYPE = 34098, 2
PAGE_SIZE, PAUSE = 100, 2.0

UA = "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/141.0.0.0 Safari/537.36"

def gis(start):
    q = {"al": 1, "ord": "redfin-recommended-asc", "sf": "1,2,3,5,6,7", "status": 9, "uipt": "1,2,3,4,5,6,7,8", "v": 8,
         "region_id": REGION_ID, "region_type": REGION_TYPE, "num_homes": PAGE_SIZE, "start": start}
    req = urllib.request.Request("https://www.redfin.com/stingray/api/gis?" + urllib.parse.urlencode(q),
                                 headers={"User-Agent": UA, "Accept": "application/json", "Accept-Encoding": "gzip"})
    try:
        with urllib.request.urlopen(req, timeout=60) as r:
            body = r.read()
            if r.headers.get("Content-Encoding") == "gzip":
                body = gzip.decompress(body)
    except urllib.error.HTTPError as e:
        print("HTTP", e.code, "- use the browser block instead")
        return []
    text = body.decode("utf-8")
    if not text.startswith("{}&&"):
        print("unexpected body:", text[:120])
        return []
    return json.loads(text[4:])["payload"].get("homes", [])

homes, seen = [], set()
for page in range(50):
    batch = gis(page * PAGE_SIZE)
    homes += [h for h in batch if h.get("propertyId") not in seen]
    seen.update(h.get("propertyId") for h in batch)
    print(f"page {page + 1}: {len(batch)} homes, {len(homes)} unique")
    if len(batch) < PAGE_SIZE:
        break
    time.sleep(PAUSE)

v = lambda x: x.get("value") if isinstance(x, dict) else x
for h in homes[:3]:
    ll = v(h.get("latLong")) or {}
    print(v(h.get("streetLine")), h.get("zip"), v(h.get("price")), h.get("beds"), h.get("baths"), v(h.get("sqFt")),
          h.get("mlsStatus"), ll.get("latitude"), ll.get("longitude"), "https://www.redfin.com" + h.get("url", ""))
```

Observed output:

```text
page 1: 100 homes, 100 unique
page 2: 100 homes, 200 unique
page 3: 100 homes, 300 unique
page 4: 31 homes, 331 unique
2704 Little John Ln 78704 775000 3 2.0 1745 Active 30.2320405 -97.7502139 https://www.redfin.com/TX/Austin/2704-Little-John-Ln-78704/home/31631298
1818 Margaret St 78704 2995000 3 4.0 2767 Coming Soon 30.2581798 -97.7689286 https://www.redfin.com/TX/Austin/1818-Margaret-St-78704/home/31205095
...
```

## Fields

Search rows (`/stingray/api/gis` -> `payload.homes[]`). Many numeric fields are wrapped as `{"value": X, "level": 1}`; a wrapper with no `value` means hidden (seen for 17 Austin prices).

| Output field | Source in each home | Example |
|---|---|---|
| property_id | `propertyId` | `32017342` |
| listing_id | `listingId` | `224635318` |
| mls_id | `mlsId.value` | `"8807705"` |
| address | `streetLine.value` (includes unit, e.g. `#1303`) | `"3305 Tavistock Dr"` |
| city / state / zip | `city`, `state`, `zip` | `"Austin"`, `"TX"`, `"78748"` |
| price | `price.value` (USD int) | `599900` |
| beds / baths | `beds`, `baths` (also `fullBaths`, `partialBaths`) | `3`, `2.5` |
| sqft | `sqFt.value` | `2190` |
| lot_sqft | `lotSize.value` | `6638` |
| year_built | `yearBuilt.value` | `2007` |
| price_per_sqft | `pricePerSqFt.value` | `274` |
| hoa_month | `hoa.value` | `48` |
| days_on_market | `dom.value` | `1` |
| status | `mlsStatus` | `"Active"`, `"Coming Soon"`, `"Price Change"` |
| neighborhood | `location.value` | `"Bauerle Ranch"` |
| lat / lon | `latLong.value.latitude/longitude` | `30.1699391`, `-97.8444657` |
| url | `"https://www.redfin.com" + url` | `.../home/32017342` |
| (not flattened) | `listingRemarks`, `keyFacts[].description`, `listingTags[]`, `sashes[].sashTypeName`, `numPictures`, `stories`, `isNewConstruction`, `uiPropertyType`, `timeZone` | `"Cowboy pool"`, `"NEW ROOF"`, `"New"` |

Detail page:

| Field | Source | Example |
|---|---|---|
| price, availability | JSON-LD `offers.price`, `offers.availability` | `599900`, `InStock` |
| address, geo | JSON-LD `mainEntity.address`, `mainEntity.geo` | PostalAddress dict |
| home_type | JSON-LD `mainEntity.accommodationCategory` | `"Single Family Residential"`, `"Condominium"` |
| beds, baths, sqft, year_built | JSON-LD `numberOfBedrooms`, `numberOfBathroomsTotal`, `floorSize.value`, `yearBuilt` | `3`, `2.5`, `2190`, `2007` |
| photos | JSON-LD `mainEntity.image[].url` | 36 URLs, ~2172x1448 |
| description, date_posted | JSON-LD `description`, `datePosted` | full remarks text |
| status | aboveTheFold `addressSectionInfo.status.displayValue` | `"Active"` |
| lot_sqft, full/half baths | aboveTheFold `lotSize`, `numFullBaths`, `numPartialBaths` | `6638`, `2`, `1` |
| mls_details | belowTheFold `amenitiesInfo.superGroups[].amenityGroups[].amenityEntries[]` | `{"Parking": ["# of Parking (Total): 4", ...]}` |

## Gotchas

- Do not `return fetch(...)` inside one `js()` call for big responses: a 350-home `gis` call usually finished in 1-4 s but once took longer, and `js()` raised `Runtime.evaluate timed out after 5s waiting for the daemon` mid-crawl. The `rf()` helper above starts the fetch, stores the text on `window`, and polls, which fixed it.
- Every `/stingray` JSON response starts with `{}&&`. Strip 4 characters before `json.loads`; anything else in that position means you got an error page.
- `page_number` does nothing: `page_number=2` returned the same first ids as page 1. Use `start=<offset>` (that is the parameter the site itself sends, as `start=0`). With `start` stepping by 350, Austin gave 5,795 unique ids with zero repeats.
- `num_homes` is the page size. 350 is what the site sends; 500 and 2,000 also returned that many in a single call, but big responses are slow and heavy, so keep 350 and page.
- `status=9`, `status=1` and `status=130` returned the identical 331 homes for ZIP 78704 (Active, Coming Soon, New). Results were only for-sale/coming-soon; no Pending or Sold rows appeared. Do not rely on `status` to switch to sold homes.
- Price filters: `min_price` / `max_price` alone were ignored (prices from 106,900 to 14,699,000 came back for a 500k-600k request). They were applied when sent together with the extra `market=austin&mpt=99` parameters the site's own filter URL produced. Simplest: fetch everything, then filter the rows in Python.
- `gis-aggregates` gives the site's headline count (`numProperties`, `numEarlyAccess`) and min/max/avg price for the same query; use it to sanity-check how many rows you should expect. It returned a count for the Austin city (5,793 + 120) and Travis County (9,474 + 145) but came back with `resultCode: 0, "Success"` and no payload for ZIP 78704, so treat it as optional.
- ZIPs and neighborhoods have their own Redfin ids. Searching ZIP 78704 needs `region_id=34098, region_type=2`; neighborhood "East Austin" is `351579` with `region_type=1` (773 homes), and the same id with the city type returned an unrelated 3,169-home set. Always resolve via autocomplete.
- City regions follow Redfin's boundary, not the mailing city: the Austin result included Del Valle (148), Pflugerville (74), Manor (30) and Cedar Park (23) addresses.
- Some listings have `Undisclosed address` (18 in Austin) and no street line; lat/lon was still present.
- Data-sharing restriction is per-MLS: the gis-csv download (see below) states "In accordance with local MLS rules, some MLS listings are not included in the download".
- The page loads a OneTrust (cookielaw) consent script. None of the calls in this skill required dismissing anything.
- Pacing used here: 2 s between search calls, 1 s between detail API calls, 3 s between detail pages. No 429 or block appeared at that rate (17 search calls back to back, 10 detail pages).
- Selector drift: none of the code depends on CSS classes. The listing cards are rendered HTML with changing class names; prefer the JSON above.

## What does not work

- Plain HTTP (urllib, outside a browser) to `/stingray/do/location-autocomplete` and to `/stingray/api/home/details/aboveTheFold` / `belowTheFold`: 403 from CloudFront ("The request could not be satisfied"). The same calls returned 200 from inside a Redfin tab.
- Plain HTTP to a listing detail page: 202 with an empty body (a bot-check interstitial, no data). Search HTML pages (`/city/...`, `/zipcode/...`) returned 200 once and then also 202 empty on a later request (`/zipcode/78704/page-2`). Use the browser for any HTML.
- Plain HTTP with the bare `Mozilla/5.0` UA to `/city/30818/TX/Austin`: 403.
- Search-page HTML is not a good bulk source anyway: a page carries one JSON-LD pair per card (41 on `/city/30818/TX/Austin/page-2` in the browser) (`SingleFamilyResidence` with address, geo, `numberOfRooms`, `floorSize`, plus a `Product` with `offers.price`), but no baths and no status.
- `/stingray/api/gis-csv` (same query string) returned 200 `text/csv` with ADDRESS, PRICE, BEDS, BATHS, SQUARE FEET, STATUS, URL, LATITUDE, LONGITUDE columns from inside the tab, but warns that some MLS listings are excluded. The JSON endpoint is the more complete source.
