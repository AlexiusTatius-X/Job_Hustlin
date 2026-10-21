# JSON listing endpoints (verified 2026-10-21)

These are GET endpoints, fetched with `firecrawl_scrape` and `formats: ["markdown"]`
(the JSON arrives inside a fenced block in `markdown`). Large responses get saved to a
file; parse them with `python3` via Bash rather than reading them into context.

## amazon-json (Amazon)
`https://www.amazon.jobs/en/search.json?category%5B%5D=software-development&country=IND&result_limit=50&sort=recent`
- `jobs[]` has `title`, `job_path` (job URL = `https://www.amazon.jobs` + `job_path`),
  `normalized_location`, `posted_date`, `id_icims`, and **`basic_qualifications`** inline.
- `basic_qualifications` is complete JD evidence for experience: parse "N+ years". Keep only
  N <= 2 (Amazon SDE I roles say "1+ years"). No JD scrape is needed for Amazon; record
  these as `inline_JD_used`. Most "Software Development Engineer" (no "I") roles are 3+ years.
- Reject `is_intern: true` and titles with "Manager", "II", "III", "Sr", "Senior".

## eightfold-json (Microsoft)
`https://apply.careers.microsoft.com/api/pcsx/search?domain=microsoft.com&query=software%20engineer&location=India&sort_by=timestamp`
- `data.positions[]` (10 newest) has `name`, `positionUrl` (job URL =
  `https://apply.careers.microsoft.com` + `positionUrl`), `locations`, `department`,
  `workLocationOption`, `postedTs`. No JD inline: scrape unseen JD URLs.
- Pre-filter: drop names with Senior/Principal/Manager/Director/Applied AI/Scientist and
  department "Applied Sciences". `workLocationOption: "remote"` → reject.
- Paging/seniority query params are ignored by this API; one page per run is it.

## smartrecruiters (Swiggy)
`https://api.smartrecruiters.com/v1/companies/swiggy/postings?limit=100&country=in&q=engineer`
- `content[]` has `name`, `id`, `location.city`, `releasedDate`. Job URL =
  `https://jobs.smartrecruiters.com/<company>/<id>`. No JD inline: scrape unseen JD URLs.

## rippling-json (Rippling)
`https://ats.rippling.com/api/v2/board/rippling/jobs`
- `items[]` has `name`, `url`, `department.name`, `locations[].country`. Keep only
  `countryCode == "IN"`. Scrape unseen JD URLs.

## Greenhouse
`https://boards-api.greenhouse.io/v1/boards/<slug>/jobs` — see `ats-greenhouse.md`.

## Lever
`https://jobs.lever.co/<slug>` (HTML board; titles + location + link per row, far smaller
than the 1 MB `api.lever.co/v0/postings/<slug>?mode=json` response). Scrape unseen JD URLs.
