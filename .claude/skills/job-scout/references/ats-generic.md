# ATS pattern: Generic / custom (fallback)

Use this for any company whose `ats` is `custom`, `oracle`, `successfactors`,
`smartrecruiters`, `ashby`, `icims`, `eightfold`, an unknown value, or one marked
`# verify` in `companies.yaml` — and as the fallback whenever a specific ATS pattern
fails. It relies on Firecrawl's general capabilities rather than a known API.

## Strategy (in order)

> Apply `references/firecrawl-tuning.md` throughout: change-tracking on the listing to
> short-circuit unchanged sites, `maxAge` cache on JD scrapes, minimal `formats`.

### A. Map the careers site (filtered), then scrape only new listings
1. Firecrawl `map` the `careers_url` **with `search: "software engineer"`** (and, if the
   site needs it, a city term) to enumerate just the relevant job URLs (~1 credit) instead
   of scraping the whole page. Keep those that look like job pages (contain `job`,
   `careers`, `search`, `role`, `opening`, or a country/city).
2. Dedup the mapped URLs vs the `Seen` tab **first**, then `scrape` only the unseen JDs
   (with `maxAge` per the tuning cheatsheet). When you do scrape the listing page itself,
   add the `changeTracking` git-diff format so an unchanged page skips the company.

### B. If mapping is unhelpful, use site-scoped search
Run Firecrawl `search` queries scoped to the careers domain, e.g.:
```
site:<careers_domain> software engineer India
site:<careers_domain> SDE Bangalore OR Hyderabad OR Noida OR Gurgaon
site:<careers_domain> software developer 0-2 years India
```
Also try the company's job search URL with query params when the site supports them
(e.g. `?keywords=software%20engineer&location=India`).

### C. Extract structured rows
Use Firecrawl `extract` with a schema like:
```json
{
  "jobs": [{
    "role_title": "string",
    "location": "string",
    "experience": "string",
    "job_url": "string",
    "job_id": "string",
    "posted_date": "string"
  }]
}
```
Open each job detail page to fill `experience` and confirm it is a core software-dev role.

## Known custom sites — quick hints
- **Google** (`google.com/about/careers`): search results page supports `?q=software%20engineer&location=India`.
- **Amazon** (`amazon.jobs`): `amazon.jobs/en/search?base_query=software+development+engineer&loc_query=India`.
- **Apple** (`jobs.apple.com/en-in`): filter by India; titles are often just "Software Engineer" — classify by JD.
- **Microsoft** (`careers.microsoft.com`): search "Software Engineer" + location India; profession = Software Engineering.
- **Meta** (`metacareers.com/jobs`): filter location = India, teams = engineering.
- **Oracle / iCIMS / SmartRecruiters / Eightfold**: these have listing pages that
  Firecrawl `extract` handles well; scope the query to India + SDE terms.

## Filter + capture
Apply the same rules as every ATS: India on-site, SDE-family role by JD content, 0–2 yrs,
exclude internships/ML/DS/research. Capture `company, role_title, location, experience,
job_url, job_id, posted_date`.

## After a `# verify` company succeeds
Note in the run summary which approach worked (e.g. "Razorpay → Lever API confirmed") so
`companies.yaml` can be corrected from `custom`/`# verify` to the real ATS next time.
