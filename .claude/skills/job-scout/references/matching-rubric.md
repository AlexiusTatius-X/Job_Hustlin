# Matching rubric

How to decide whether a posting is a keeper and how to score it. Apply **after** you have
the job's title, location, experience, and JD text. When in doubt, read the JD — classify
by **content**, not the title string.

## Step 1 — Hard filters (reject if any fails)

A posting must pass ALL of these or it is discarded (never logged, never emailed):

1. **Core software-development role.** The day-to-day is building/shipping software.
   - ACCEPT titles: Software Development Engineer, SDE, SDE-1/SDE I, Software Engineer,
     SWE / SWE I, Software Engineer I, Software Developer, Frontend/Front-End Engineer,
     Backend/Back-End Engineer, Full-Stack Engineer, UI Engineer.
   - ACCEPT odd/ambiguous titles (e.g. "Member of Technical Staff", Apple's generic
     "Software Engineer", "Engineer, Platform") **only if** the JD responsibilities are
     core software development.
   - REJECT category (even if the title says "Engineer"): Machine Learning / AI Engineer,
     Applied Scientist, Research Scientist/Engineer, Data Scientist, Data Analyst,
     Data Engineer, MLOps, Deep Learning, Computer Vision/NLP research.
   - REJECT non-dev: PM/TPM, Designer, SRE-only, QA/SDET-only, Support, Solutions/Sales,
     Security-only, DevOps-only.

2. **Experience within 0–2 years.**
   - REJECT if the JD requires **> 2 years** (e.g. "3+ years", "minimum 4 years", "5-7 yrs").
   - ACCEPT explicit 0, 0–1, 0–2, 1, 1–2, 2 years, "new grad", "entry level", "campus".
   - If experience is **unstated** but the role is clearly junior/new-grad, ACCEPT.
   - If unstated and the seniority is ambiguous (no signal), ACCEPT but score freshness
     lower and mark `experience = "not stated"`.

3. **On-site, India.**
   - ACCEPT locations physically in India.
   - REJECT Remote (any), and any location outside India.
   - Hybrid that is India-based on-site presence is acceptable; fully-remote is not.

4. **Not an internship.** Reject intern / co-op / trainee-only / apprenticeship postings.

## Step 2 — Fit score (0–10, rule-based, NO résumé)

Sum the four components; clamp to 0–10.

### Title match (0–4)
- 4 — Title explicitly SDE/SDE-1/SWE/Software Engineer I/Software Developer.
- 3 — Frontend/Backend/Full-Stack Engineer (clear software-dev specialization).
- 2 — Generic "Software Engineer" / "Engineer" confirmed as core dev by JD.
- 1 — Odd title (e.g. MTS) classified-in via JD content.
- 0 — Borderline but still passes hard filters.

### Experience fit (0–3)
- 3 — Explicit 0–1 years / new grad / entry level.
- 2 — Explicit 1–2 years.
- 1 — Up to 2 years but near the ceiling, or "0–2" range.
- 0 — Experience not stated.

### Location priority (0–2)
- 2 — Bengaluru/Bangalore, Hyderabad, Noida, or Gurgaon/Gurugram.
- 1 — Other Indian city (Pune, Chennai, Mumbai, Delhi, etc.).
- 0 — India but city unclear.

### Freshness (0–1)
- 1 — Posted within the last 7 days (or clearly "new"/"just posted").
- 0.5 — Posted 8–14 days ago.
- 0 — Older than `freshness_days`, or no date available.

## Step 3 — Output
Attach `fit_score` (one decimal) to each kept posting and sort descending. Ties break by
freshness, then by location priority.

## Edge cases
- **Multiple India locations on one posting:** keep it once; use the highest-priority city
  for the location-priority score; list the cities in the `location` field.
- **Same role, different req IDs / cities as separate postings:** treat each distinct
  `job_url` as its own row (dedup is by URL, not by title).
- **JD in a language other than English:** translate mentally; the rules still apply.
