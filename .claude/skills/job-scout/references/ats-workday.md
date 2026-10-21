# ATS pattern: Workday

Workday career sites live on `*.myworkdayjobs.com` (e.g.
`nvidia.wd5.myworkdayjobs.com/NVIDIAExternalCareerSite`). They are JS-heavy.

## 0. What actually works (verified 2026-10-21)
The `/wday/cxs/.../jobs` JSON endpoint below needs a **POST**, and `firecrawl_scrape` can
only GET, so it does not work through this connector. Use the rendered search page:

```json
{ "url": "https://<tenant>.<dc>.myworkdayjobs.com/<SITE>?q=software%20engineer%20India",
  "formats": ["markdown", "changeTracking"], "onlyMainContent": true, "waitFor": 6000 }
```

Without `waitFor` the page returns only "Search for Jobs page is loaded / Loading" and no
rows. With it, each row has the title, job link, `India-<City>` in the URL path, "Posted N
Days Ago" and the req id. Drop titles containing Senior/Sr/Staff/Principal/Lead/Manager
before opening JDs. A redirect to "Workday is currently unavailable" means the tenant,
data-center (`wdN`) or site path is wrong — not an outage.

Sections 1–3 below are kept for reference only.

## 1. Identify the tenant + site path
From `careers_url` like `https://<tenant>.<dc>.myworkdayjobs.com/<SITE>`:
- `tenant` = first sub-domain label (e.g. `nvidia`)
- `dc` = data-center label (e.g. `wd5`)
- `SITE` = the career-site path segment (e.g. `NVIDIAExternalCareerSite`)

If any part is unknown, use Firecrawl `map`/`search` on the `careers_url` to discover the
real site path, then continue.

## 2. Query the JSON jobs endpoint
POST-style endpoint (Firecrawl `scrape` the URL; if a body is needed, use the generic
listing scrape instead):

```
https://<tenant>.<dc>.myworkdayjobs.com/wday/cxs/<tenant>/<SITE>/jobs
```

Typical JSON body:
```json
{
  "appliedFacets": { "locationCountry": ["India"] },
  "limit": 20,
  "offset": 0,
  "searchText": "software engineer"
}
```
Run the search once per relevant term: `software engineer`, `software developer`,
`SDE`, `frontend engineer`, `backend engineer`, `full stack engineer`. Page with
`offset` until results run out or the freshness window is exceeded.

Each result item usually has: `title`, `externalPath`, `locationsText`, `postedOn`,
`bulletFields` (often the req id). Build the job URL as:
```
https://<tenant>.<dc>.myworkdayjobs.com/<SITE><externalPath>
```

## 3. If the JSON path fails
Fall back to Firecrawl `extract` on the rendered listing URL with a location filter for
India in the query, then `scrape` each job detail page.

## 4. Fields to capture
- `role_title` ← `title`
- `location` ← `locationsText` (keep only India locations)
- `job_url` ← constructed as above
- `job_id` ← req id from `bulletFields` if present
- `posted_date` ← `postedOn` (e.g. "Posted 3 Days Ago")
- `experience` ← open the detail page and read the JD (Workday list items rarely include it)

## 5. Notes
- Filter to India at query time via `locationCountry` / `locationRegion` facets when
  available — cuts noise massively.
- Read the JD for experience + to confirm it's a core SDE role (not ML/DS) per the rubric.
