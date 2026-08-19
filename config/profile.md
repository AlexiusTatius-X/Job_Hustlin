# Search Profile

Static configuration read by the `job-scout` skill on every run. Edit values here;
do not hardcode them in the skill.

## Notifications

- **notify_email:** `oniidaddy881@gmail.com`
- **email_when_empty:** `true`   # still send a short "no new jobs" email so you know it ran

## State store (Google Sheets)

- **sheet_url:** `https://docs.google.com/spreadsheets/d/1PsxcTYjLEbmK2MfRKmQBh9l9BZhhJZEC5ARx8w9l9TA/edit`
- **sheet_id:** `1PsxcTYjLEbmK2MfRKmQBh9l9BZhhJZEC5ARx8w9l9TA`

## Schedule / time

- **timezone:** `Asia/Kolkata` (IST)
- **cadence:** 3 runs per day (exact times set manually in the routine UI)
- **week_anchor_monday:** `2026-08-17`
  # Monday used to compute sequential week numbers: NN = floor((thisMonday - anchor)/7) + 1.
  # Set this to the Monday of (or before) your first real run. Default = week of first build.

## Role criteria (STRICT)

**Target titles (include):**
- Software Development Engineer / SDE / SDE-1 / SDE I
- Software Engineer / SWE / SWE I / Software Engineer I
- Software Developer
- Frontend Engineer / Front-End Engineer / UI Engineer
- Backend Engineer / Back-End Engineer
- Full-Stack Engineer
- Member of Technical Staff (MTS) — **only if** the JD is core software development
- Any other title that, by **job-description content**, is a core software-development role

**Experience:** required experience must fall within **0–2 years**.
- Reject if the JD requires **more than 2 years** (e.g. "3+ years", "4–6 years").
- Allow if unstated but clearly junior / new-grad / entry-level.

**Exclude (hard filters — never include):**
- Internships / interns / co-op / trainee-only programs
- Machine Learning / AI Engineer, Applied Scientist, Research (Scientist/Engineer)
- Data Scientist, Data Analyst, Data Engineer
- AI/ML-software, MLOps, Deep Learning roles
- Roles requiring > 2 years of experience
- Non-engineering roles (PM, TPM, designer, SRE-only, QA-only, support, sales)

**Location (STRICT):**
- **On-site, India only.**
- Priority cities (rank higher): **Bengaluru/Bangalore, Hyderabad, Noida, Gurgaon/Gurugram**.
- Allow other Indian cities but flag them lower.
- **Reject:** Remote, Hybrid-remote outside India, and any non-India location.

## Fit score (rule-based — NO résumé)

Score each passing job 0–10 (see `references/matching-rubric.md` for the exact rubric).
Components: title-match strength, experience fit, location priority, freshness.

## Per-run limits (politeness / cost control)

- **max_companies_per_run:** `all`   # ~45; reduce if you hit Firecrawl/route limits
- **max_new_jobs_emailed:** `40`
- **freshness_days:** `14`            # ignore postings clearly older than this when a date is available
