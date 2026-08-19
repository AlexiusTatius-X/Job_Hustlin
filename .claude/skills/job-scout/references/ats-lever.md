# ATS pattern: Lever

Lever boards live at `jobs.lever.co/<org>`. Like Greenhouse, Lever exposes a **public
JSON API** — prefer it over scraping.

## 1. Find the org slug
From `careers_url`, extract the Lever org token (e.g. `razorpay`). If the company uses a
custom careers domain, use Firecrawl `map`/`search` to find the `jobs.lever.co/<org>`
board and read `<org>`.

## 2. Public JSON API (preferred)
```
https://api.lever.co/v0/postings/<org>?mode=json
```
Returns an array of postings, each with: `text` (title), `hostedUrl`, `categories`
(`location`, `team`, `commitment`), `createdAt`, `id` (job_id), and
`descriptionPlain` (JD text for experience parsing).

Optional server-side filters you can append:
```
&location=Bangalore   &commitment=Full-time   &team=Engineering
```

Firecrawl `scrape` the API URL (returns JSON).

## 3. Filter
- Keep postings whose `categories.location` is an Indian city; reject remote/non-India.
- Keep `categories.commitment` = Full-time; drop internships (commitment "Intern").
- Keep SDE-family titles; confirm via `descriptionPlain` it's core software dev, not ML/DS.
- Parse `descriptionPlain` for experience; reject ">2 years".

## 4. Fields to capture
- `role_title` ← `text`
- `location` ← `categories.location`
- `job_url` ← `hostedUrl`
- `job_id` ← `id`
- `posted_date` ← `createdAt` (epoch ms)
- `experience` ← parsed from `descriptionPlain`

## 5. If the API fails
Fall back to Firecrawl `extract` on `jobs.lever.co/<org>`, then scrape each posting page.
