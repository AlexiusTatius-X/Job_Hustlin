# Firecrawl credit-tuning cheatsheet

Read this before scraping. These are the levers that cut Firecrawl credits (and run time)
without missing jobs. Ordered by impact for our 3x/day cadence. Credit facts are from the
Firecrawl v2 docs (change-tracking billing, scrape `maxAge` cache, `/map`).

## 1. `changeTracking` on the LISTING — the steady-state saver (FREE)

Basic and `git-diff` change tracking cost **no extra credits** (only `json` mode costs 5).
Snapshots are stored **server-side by Firecrawl, scoped to your team, and never expire** —
so this works even though the routine's repo is read-only across runs (no state to keep).

When you fetch a company's listing (careers page or a GET-able ATS JSON/API URL), request:

```json
{
  "url": "<careers_or_api_url>",
  "formats": ["markdown", { "type": "changeTracking", "modes": ["git-diff"], "tag": "jobscout" }],
  "onlyMainContent": true
}
```

Then branch on `changeTracking.changeStatus`:

| `changeStatus` | What it means            | Action                                                                 |
| -------------- | ------------------------ | ---------------------------------------------------------------------- |
| `"same"`       | Page identical to last run | **Skip the whole company this run.** No map, no JD scrapes, no LLM pass. |
| `"new"`        | First scrape ever        | Process the full listing (collect all candidate rows), then dedup vs `Seen`. |
| `"changed"`    | Page differs             | Read `diff.text`; the **added (`+`) lines hold the new job rows** — take candidate URLs from those, then dedup vs `Seen`. |
| `"removed"`    | Listing gone             | Log the error, continue.                                               |

Why it helps: across 3 daily runs, most careers pages are unchanged between runs, so a
single cheap listing scrape ends the work for that company. On `"changed"`, the git-diff
hands you exactly the new rows, so you never re-enumerate the whole page.

Notes:
- `markdown` **must** be in `formats` (change tracking compares markdown).
- `changeTracking` **bypasses the cache** (`maxAge` is ignored on these calls) — that's
  fine; the listing scrape is 1 cheap call and it gates everything downstream.
- Keep `onlyMainContent`, `includeTags`/`excludeTags` **consistent** run-to-run, or diffs
  become unreliable. Use one fixed `tag: "jobscout"`.

## 2. `maxAge` cache on JD (detail) scrapes — cheaper reads

A job description is immutable once posted, so allow a long cache window on JD scrapes:

```json
{ "url": "<jd_url>", "formats": ["markdown"], "onlyMainContent": true, "maxAge": 604800000 }
```

`maxAge` = 7 days (ms). If Firecrawl has a fresh-enough snapshot it returns the cached
copy at reduced cost instead of a full re-scrape. `Seen`-dedup already prevents most
re-scrapes; `maxAge` covers the rest (e.g. a JD that appears under two cities in one run).
Default `maxAge` is 2 days; set `0` only when you truly need a fresh fetch.

## 3. `/map` with a `search` filter instead of scraping a heavy listing

For `custom`/unknown sites where the listing page is large, prefer `map` to enumerate only
relevant job URLs (~1 credit) rather than scraping the full page:

```json
{ "url": "<careers_url>", "search": "software engineer", "limit": 100 }
```

Returns candidate job URLs matching the text. Dedup those vs `Seen` **first**, then scrape
only the unseen JDs. (ATS JSON APIs — Greenhouse/Lever/Workday — already return listings
cheaply and often include the JD inline, so skip `map` for those.)

## 4. Minimize the payload on every scrape (cuts tokens + time)

- Request only the formats you use — usually just `["markdown"]` (add `"links"` only when
  you need to harvest URLs). Avoid `html`, `rawHtml`, `screenshot`, and `json`/`extract`
  unless necessary (`json` extraction costs more).
- Keep `onlyMainContent: true` and use `excludeTags` (nav/footer/cookie banners) so the
  model reads less boilerplate.

## 5. Prefer the ATS JSON API over scraping (already the default)

Greenhouse (`?content=true`), Lever (`descriptionPlain`), and Workday detail calls return
the JD **inline** with the listing — zero extra JD scrapes. Always try the API path first
(see the per-ATS reference files); fall back to scrape/map only for `custom` sites.

## 6. Batch the scrapes — throughput, NOT credits

`/batch/scrape` costs exactly **1 credit per URL**, the same as scraping each URL alone
(verified: 36 URLs → 36 credits). It gives **no credit discount**, but it runs the URLs
concurrently in one job and accepts all scrape options (`changeTracking`, `maxAge`,
`onlyMainContent`). Use it to cut tool-call round-trips and latency:
- Batch all **due listing pages** into one call with the `changeTracking` git-diff format.
- Batch all **unseen JD** detail pages into one call with `maxAge`.
Only a throughput win — the credit total is unchanged, so tiering + change-tracking remain
the real savers.

## 7. Avoid the credit-expensive features

- **`/agent`** (autonomous extraction) can spend up to 2,500 credits per call — do **not**
  use it for routine scouting.
- **`json` / `extract` formats** and **`json`-mode change tracking** cost extra (5 credits/
  page for json change tracking) — use plain `git-diff` and read the rows yourself.
- **Enhanced mode, stealth proxies, PII redaction** all add credits and are unnecessary
  for public careers pages — leave them off.

## Evaluated but NOT adopted: `/monitor`

Firecrawl's hosted **Monitor** schedules recurring checks and emails/webhooks on change.
It was considered and rejected for this skill because:
- **No credit savings**: a scrape monitor is **1 credit/URL/check** — identical to our
  `changeTracking` listing scrape. It relocates the work, it doesn't cheapen it.
- It **can't** run our domain logic (SDE-by-JD classification, 0–2yr filter, on-site-India
  rule, Google-Sheet dedup, fit ranking) — its `goal` judge is generic and costs +1 credit/
  changed page.
- It notifies by **webhook**, which a time-scheduled Claude Routine can't act on.
Keep the logic in the skill; use `changeTracking` (lever 1) for the same credit floor.

## Steady-state effect

Tier gating (A/B/C) limits *which* companies run; change-tracking (lever 1) then makes each
*unchanged* company cost a single cheap listing call with **no JD scrapes**. Net: on a
typical run only the handful of companies that actually posted something incur JD-scrape
credits.
