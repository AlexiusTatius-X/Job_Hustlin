# Routine prompt

Paste the block below into the **Instructions** field when creating the routine at
`claude.ai/code/routines`. Select this repository, and include the **Firecrawl**,
**Gmail**, and **Google Sheets** connectors. Add a **Schedule** trigger with 3 daily
times in IST (your choice, e.g. 09:00 / 14:00 / 19:00).

---

You are running the **job-scout** routine from this repository.

Load and follow the skill at `.claude/skills/job-scout/SKILL.md` exactly. Read
`config/profile.md` and `config/companies.yaml` first, then execute the skill's full
workflow for this run:

1. Determine the current date/time in IST and this week's Monday.
2. Open the Google Sheet named in `config/profile.md`; ensure the current week tab exists
   (create it if missing) per `references/sheet-format.md`.
3. For every company in `config/companies.yaml`, use the Firecrawl connector to find
   currently-open software-engineering roles, using the matching `references/ats-*.md`
   pattern (fall back to `references/ats-generic.md` when needed).
4. Filter strictly with `references/matching-rubric.md`: keep only core software-developer
   (SDE/SWE) roles requiring 0–2 years, on-site in India (prefer Bengaluru, Hyderabad,
   Noida, Gurgaon). Exclude internships, ML/AI, data science, research, and anything
   requiring more than 2 years.
5. De-duplicate against jobs already in the sheet (Option A: log each posting once).
6. Rank the new jobs by the rule-based fit score.
7. Append the new jobs to today's section of the current week tab.
8. Email the digest to the address in `config/profile.md` via Gmail, formatted per
   `references/email-format.md`.
9. End with a short summary: companies scanned, errors, found → kept → new, rows appended,
   email sent.

Only use the Firecrawl, Google Sheets, and Gmail connectors plus this repo's files. Never
fabricate a job, URL, or company — every logged row must come from a real posting you
actually fetched this run. If a company fails, record the error and continue.

---

## Notes
- The routine grants every tool of the included connectors with no per-run approval, so
  keep the connector set to exactly Firecrawl + Gmail + Google Sheets.
- Times are entered in your local zone (IST) and converted automatically. Minimum interval
  between scheduled runs is 1 hour; 3×/day is well within limits.
- After creating the routine, click **Run now** once and open the session transcript to
  verify: Firecrawl calls happened, the week tab was created/updated, and the email sent.
