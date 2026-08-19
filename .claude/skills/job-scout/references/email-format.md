# Email digest format

Send via the Gmail connector to `notify_email` (from `config/profile.md`). One email per
run, covering only the **new** jobs found in that run.

## Subject
```
[Job Scout] N new SDE roles — 25 Aug, 09:00 IST
```
- `N` = number of new jobs in this run. If 0 and `email_when_empty` is true:
  `[Job Scout] No new SDE roles — 25 Aug, 09:00 IST`.

## Body (HTML preferred, plain-text fallback)

Header line:
```
25 Aug 2026 · 09:00 IST run · N new jobs · Companies scanned: 45 · Errored: 2
```

Then the new jobs **grouped by company**, companies ordered by their top job's fit score,
and jobs within a company sorted by fit descending. For each job:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Microsoft
  • Software Engineer II — Hyderabad
    Exp: 0–2 yrs · Fit: 8.5
    Apply: https://careers.microsoft.com/...

  • Software Engineer — Noida
    Exp: 1 yr · Fit: 7.0
    Apply: https://careers.microsoft.com/...
```

At the bottom:
```
Full log: <sheet_url>   (tab: Week 01 · 25–31 Aug)
Companies that errored this run: Optiver, WorldQuant
```

## Rules
- Only include jobs newly added in **this** run (already de-duped against the sheet).
- Respect `max_new_jobs_emailed`; if exceeded, include the top-N by fit and add a line
  "…and M more in the sheet."
- Every job must have a working direct **Apply** link.
- Keep it skimmable: company → bulleted roles → one apply link each. No walls of text.
- If 0 new jobs and `email_when_empty` is false, do not send.
