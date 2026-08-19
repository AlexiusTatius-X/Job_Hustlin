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

### 2. Load the Google Sheet state
- Open the spreadsheet by `sheet_id` from `profile.md`.
- Ensure the current **week tab** exists; if missing, create it with the header block
  (see `references/sheet-format.md`).
- Read the **existing rows in the current week tab** AND, for dedup safety, the previous
  week tab if it exists. Build a set of already-seen keys = normalized `job_url` (and
  `job_id` when present). This set drives Option-A dedup: a posting already in the sheet
  is never logged or emailed again.

### 3. Scrape each company
For each company in `companies.yaml`:
- Pick the extraction approach from its `ats` type and read the matching reference:
  - `workday` → `references/ats-workday.md`
  - `greenhouse` → `references/ats-greenhouse.md`
  - `lever` → `references/ats-lever.md`
  - anything else (`custom`, `oracle`, `successfactors`, `smartrecruiters`, `ashby`,
    `icims`, `eightfold`, or an unknown/`# verify` value) → `references/ats-generic.md`
- Use Firecrawl to get the current software-engineering openings: search/map to find the
  right listing URL(s), then `scrape`/`extract` job rows. Prefer structured `extract`.
- If a company's `ats` is marked `# verify` or the pattern fails, fall back to
  `ats-generic.md` (Firecrawl `search` scoped to the careers domain + India + SDE terms),
  and note the working approach in the run summary so the config can be corrected later.
- Be resilient: if one company errors, record the error and continue to the next.

### 4. Classify & filter each posting
Apply `references/matching-rubric.md` strictly. Keep a posting **only if all** hold:
- It is a **core software-development** role (SDE/SWE/software developer or equivalent) —
  decide by **job-description content**, not just the title (handles odd titles like Apple's).
- Required experience is within **0–2 years** (reject ">2 years").
- Location is **on-site in India** (reject remote / non-India).
- It is **not** an excluded category (internship, ML/AI, data science, research, etc.).

For each kept posting, capture: `company, role_title, location, experience, job_url,
job_id` (if available), `posted_date` (if available).

### 5. De-duplicate (Option A)
- Drop any posting whose normalized `job_url`/`job_id` is already in the seen-set from
  step 2. A given posting is logged **once**, at first sight.
- Two **different** postings (different URLs) from the same company are **both** kept —
  they become separate rows.

### 6. Rank
- Compute a 0–10 **fit score** per remaining posting using the rule-based rubric in
  `references/matching-rubric.md` (title match, experience fit, location priority,
  freshness). Sort new jobs by fit score descending.

### 7. Append to the Google Sheet
- Under **today's day-header row** in the current week tab (create the day header if this
  is the first write of the day), append one row per new job. Follow the exact column
  order and formatting in `references/sheet-format.md`. Set `status = New`.

### 8. Email the digest
- Send an email via Gmail to `notify_email`, formatted per `references/email-format.md`:
  new jobs grouped by company, highest fit first, with title, location, experience, and a
  direct apply link. Subject line includes the count and date.
- If there are **no** new jobs and `email_when_empty` is true, send the short "no new
  jobs today" note; otherwise skip the email.

### 9. Run summary
- End with a brief transcript summary: companies scanned, companies that errored,
  postings found → kept → new, rows appended, email sent. Flag any `# verify` companies
  whose ATS you confirmed so the config can be updated.

## Guardrails
- **Never fabricate** a posting, URL, or company. If you didn't fetch it, don't log it.
- Respect `max_new_jobs_emailed` and `max_companies_per_run` from `profile.md`.
- Keep all mutable state in the Google Sheet — the repo is read-only across runs.
- Do not email anyone other than `notify_email`.
- If Firecrawl returns an error/`jobId`, retry once; if it still fails, skip that company
  and record it. Do not guess results.
