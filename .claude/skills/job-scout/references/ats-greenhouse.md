# ATS pattern: Greenhouse

Greenhouse boards live at `boards.greenhouse.io/<org>` or `job-boards.greenhouse.io/<org>`,
and many companies embed the same data on their own careers page. Greenhouse exposes a
clean **public JSON API** — always prefer it.

## 1. Find the org slug
From `careers_url`, find the Greenhouse org token (e.g. `databricks`, `mongodb`). If the
company uses a custom careers domain, use Firecrawl `map`/`search` to locate the
`greenhouse.io/<org>` board or the embed, then extract `<org>`.

## 2. Public JSON API (preferred)

Fetch metadata first; do **not** request every JD inline on a large board:

```
https://boards-api.greenhouse.io/v1/boards/<org>/jobs
```

This listing returns `title`, `absolute_url`, `location.name`, `updated_at`, and `id`
without loading every job description into the model context. Coarse-filter by role and
India location, then dedup URLs/IDs against `Seen`.

For each unseen candidate only, fetch its detail:

```
https://boards-api.greenhouse.io/v1/boards/<org>/jobs/<id>
```

The detail response supplies the JD `content` for experience and responsibility checks.

For small boards only, this combined endpoint is acceptable when the response remains
compact:

```
https://boards-api.greenhouse.io/v1/boards/<org>/jobs?content=true
```
Returns `jobs[]` with: `title`, `absolute_url`, `location.name`, `updated_at`,
`id` (job_id), and (with `content=true`) the HTML `content` (JD) for experience parsing.

Firecrawl `scrape` these API URLs (they return JSON).

## 3. Filter
- Keep jobs whose `location.name` is in India (Bengaluru/Bangalore, Hyderabad, Noida,
  Gurgaon/Gurugram, or other Indian city). Reject remote/non-India.
- Keep SDE-family titles; confirm via `content` it's core software dev, not ML/DS.
- Parse `content` for experience; reject ">2 years".

## 4. Fields to capture
- `role_title` ← `title`
- `location` ← `location.name`
- `job_url` ← `absolute_url`
- `job_id` ← `id`
- `posted_date` ← `updated_at`
- `experience` ← parsed from `content`

## 5. If the API fails
Fall back to Firecrawl `extract` on `boards.greenhouse.io/<org>` (or the embed page),
then scrape each job's detail page.
