---
name: job-scout
description: >-
  Finds new entry-level software-developer (SDE/SWE) jobs at top companies in India
  and reports them. Use when asked to scout, search, or check for new SDE/SWE/software
  developer jobs, run the job hunt, or refresh the job tracker. Scrapes each company's
  careers site via the Firecrawl connector, filters to on-site India roles requiring
  0-2 years, de-duplicates against a Google Sheet, appends new matches, and emails a
  digest. Built to run unattended from a Claude Routine, 3x/day.
---

# job-scout

Scout top companies' career portals for **new entry-level software-developer roles in
India**, log them to a Google Sheet, and email a digest. Designed to run **autonomously**
from a Claude Routine three times a day.

You have three connectors available: **Firecrawl** (web search/scrape/extract),
**Google Sheets** (state + log), and **Gmail** (digest email). Use only these plus repo
files. Never invent jobs — every row must come from a real, currently-open posting you
actually fetched.

## Inputs (read these first, every run)

1. `config/profile.md` — criteria, `notify_email`, `sheet_id`/`sheet_url`, `timezone`,
   `week_anchor_monday`, per-run limits. **All rules come from here.**
2. `config/companies.yaml` — the companies to scan and each one's `ats` type + `careers_url`.

## Workflow (per run)

Run these steps in order. Work company-by-company; independent companies may be
processed in parallel batches if that speeds things up.

### 1. Set up run context
- Compute **now** in the `timezone` from `profile.md` (IST). Record `today` (date) and
  `run_time` (HH:MM).
- Compute **this week's Monday** = the most recent Monday ≤ `today`.
- Compute the week tab name (see `references/sheet-format.md` → "Week tab naming").

### 2. Load dedup memory (the `Seen` tab)
- Open the spreadsheet by `sheet_id` from `profile.md`.
- Ensure the **`Seen`** tab exists — the all-time, long-format ledger of every job URL
  ever logged (see `references/sheet-format.md` → "The `Seen` tab"). Create it with its
  header if missing.
- Read the **`Seen` tab's `job_url` column** into an in-memory **seen-set** (normalize
  each URL). This one tab is the **only** dedup index — do NOT dedup against the weekly
  tabs. It spans all time, so a still-open job posted weeks ago is never re-logged.
- Ensure the current **week tab** exists (this is the human-readable digest log, not the
  dedup source); if missing, create it with the header block per `sheet-format.md`.

### 3. Discover listings + cheap URL dedup (per company)
Process each company in `companies.yaml`. **Do this before opening any job-description
(JD) pages** — it is what keeps credits/time low.
- **Tier gate (which companies run this time):** include a company only if its `tier`
  is due for this run, based on the IST time from step 1:
  - `tier: A` → **every run**.
  - `tier: B` → only the **day's first (morning) run** — i.e. IST hour < 12.
  - `tier: C` → only the **first run of Monday and Thursday** — IST hour < 12 **and**
    weekday ∈ {Mon, Thu}.
  Skip companies not due; they're covered on their next scheduled cadence. The rule is
  derived purely from the timestamp, so no extra state is needed.
- Read `references/firecrawl-tuning.md` once and apply its credit levers to **every**
  Firecrawl call below (change-tracking on listings, `maxAge` cache on JD scrapes,
  `map` with a `search` filter, minimal `formats`).
- Pick the extraction approach from its `ats` type and read the matching reference:
  - `workday` → `references/ats-workday.md`
  - `greenhouse` → `references/ats-greenhouse.md`
  - `lever` → `references/ats-lever.md`
  - anything else (`custom`, `oracle`, `successfactors`, `smartrecruiters`, `ashby`,
    `icims`, `eightfold`, or an unknown/`# verify` value) → `references/ats-generic.md`
- **Change-tracking short-circuit (free; do this first):** fetch the listing (careers page,
  or a GET-able ATS/API URL) with `formats: ["markdown", {type:"changeTracking",
  modes:["git-diff"], tag:"jobscout"}]` and `onlyMainContent:true`. **Batch it:** collect
  every due company's listing URL and fetch them together in **one `batch/scrape` call**
  (`maxConcurrency` left at default) instead of 16–46 separate scrapes — same credits
  (1/URL), but concurrent and one round-trip. Then, per returned page:
  - `changeStatus == "same"` → the page is identical to the last run: **skip this company
    entirely** (no map, no JD scrapes, no classification). Record it as "unchanged".
  - `changeStatus == "changed"` → take candidate rows only from the **added (`+`) lines**
    of `diff.text` (these are the new postings).
  - `changeStatus == "new"` (first ever) → process the full listing as usual.
  This is server-side state (per team, never expires), so it needs no repo storage.
  (ATS JSON APIs that need POST/params can't go in the batch — fetch those individually.)
- Fetch the **listing only** (the cheap part): the ATS JSON API, or Firecrawl `map`
  (with `search:"software engineer"`) / `search` / `scrape` on the careers page. From it
  collect candidate rows of `role_title`, `location` (if shown), `job_url`, `job_id`
  (if present). One fetch, minimal `formats`.
- Coarse pre-filter on what the listing already shows: drop obvious non-matches
  (clearly ML/DS/PM/research titles, or a shown location that is remote/outside India).
  Be permissive — the real decision happens by JD in step 4.
- **Dedup #1 — URL vs `Seen` (this is the credit saver):** drop every candidate whose
  normalized `job_url`/`job_id` is already in the seen-set from step 2. Only **unseen**
  candidates continue. On steady-state runs this eliminates almost everything, so you
  open very few JDs.
- Note: for ATS-API companies (Greenhouse `content=true`, Lever `descriptionPlain`,
  Workday detail) the JD text often comes back **inline** in the listing call — keep it,
  so step 4 needs no extra fetch for those.
- Be resilient: if one company errors, record the error and continue to the next.

### 4. Open JDs for unseen candidates only, then classify & filter
- Gather the **unseen** candidates from all companies, then fetch their JDs in **one
  `batch/scrape` call** with `formats:["markdown"]`, `onlyMainContent:true`, and
  `maxAge: 604800000` (7-day cache — a JD is immutable once posted, so a cached read is
  cheaper). Reuse any inline JD already returned in step 3 (ATS APIs) and exclude those
  from the batch. **This per-JD scrape — the only expensive step — is now paid only for
  genuinely new postings**, and batching makes the run concurrent (same 1 credit/URL).
- Apply `references/matching-rubric.md` strictly. Keep a posting **only if all** hold:
  - It is a **core software-development** role (SDE/SWE/software developer or equivalent)
    — decide by **JD content**, not just the title (handles odd titles like Apple's).
  - Required experience is within **0–2 years** (reject ">2 years").
  - Location is **on-site in India** (reject remote / non-India).
  - It is **not** an excluded category (internship, ML/AI, data science, research, etc.).
- For each kept posting, capture: `company, role_title, location, experience, job_url,
  job_id` (if available), `posted_date` (if available).

### 5. De-duplicate within this run (Dedup #2)
- Among this run's kept postings, remove any repeats by normalized `job_url` — the same
  posting can surface twice in one run (e.g. listed under two cities, or found via both
  the careers page and a web search). Log it **once**.
- (Dedup #1 in step 3 already handled repeats vs previous runs.)

### 6. Rank
- Compute a 0–10 **fit score** per remaining posting using the rule-based rubric in
  `references/matching-rubric.md` (title match, experience fit, location priority,
  freshness). Sort new jobs by fit score descending.

### 7. Append to the Google Sheet (both tabs)
- For every new job, append a row to the **`Seen`** tab — `job_url, job_id, company,
  role_title, first_seen = today` — so it is skipped by Dedup #1 on all future runs.
- Also append it under **today's day-header row** in the current **week tab** (the
  digest log; create the day header if this is the first write of the day), with
  `status = New`. Follow the exact columns/format in `references/sheet-format.md`.

### 8. Email the digest
- Send an email via Gmail to `notify_email`, formatted per `references/email-format.md`:
  new jobs grouped by company, highest fit first, with title, location, experience, and a
  direct apply link. Subject line includes the count and date.
- If there are **no** new jobs and `email_when_empty` is true, send the short "no new
  jobs today" note; otherwise skip the email.

### 9. Run summary
- End with a brief transcript summary: companies scanned, companies **skipped as unchanged**
  (change-tracking `"same"`), companies that errored, and the funnel — candidates listed →
  unseen after Dedup #1 → JDs opened → kept after filter → rows logged — plus whether the
  email was sent. Flag any `# verify` companies whose ATS you confirmed so the config can
  be updated.

## Guardrails
- **Never fabricate** a posting, URL, or company. If you didn't fetch it, don't log it.
- Respect `max_new_jobs_emailed` and `max_companies_per_run` from `profile.md`.
- Keep all mutable state in the Google Sheet — the repo is read-only across runs.
- Do not email anyone other than `notify_email`.
- If Firecrawl returns an error/`jobId`, retry once; if it still fails, skip that company
  and record it. Do not guess results.
