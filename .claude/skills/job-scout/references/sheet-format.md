# Google Sheet format

The spreadsheet (id in `config/profile.md`) is the state store + running log. It is the
**only** place mutable state lives. Use the Google Sheets connector.

## One tab per week

- A **week** = Monday 00:00 → Sunday 23:59 in IST (`Asia/Kolkata`).
- Each run computes **this week's Monday** = the most recent Monday ≤ today.
- If the tab for this week does not exist, **create it** (with the header block below),
  then append. This "Monday-anchored" rule means the correct tab is derived fresh every
  run — no counter to persist.

### Week tab naming
`Week NN · dd–dd Mon` (example: `Week 01 · 25–31 Aug`).
- `NN` = `floor((thisMonday − week_anchor_monday) / 7) + 1`, zero-padded to 2 digits.
  `week_anchor_monday` is in `config/profile.md`.
- The date range is Monday–Sunday of the week (same month shown once, e.g. `25–31 Aug`;
  if it spans months, use `29 Sep–05 Oct`).
- The **first run mid-week still uses this week's tab** — it is Week 1 (partial); the next
  Monday naturally yields Week 2.

## Tab layout

Row 1 — title/banner (merged across A:I):
```
Week 01 · 25–31 Aug 2026   (Mon–Sun, IST)
```
Row 2 — column headers:

| A | B | C | D | E | F | G | H | I |
|---|---|---|---|---|---|---|---|---|
| Date | Run (IST) | Company | Role Title | Location | Exp | Fit | Job URL | Status |

Then, for each day that has new jobs, a **day-header separator row** followed by that
day's job rows.

### Day-header separator row
Before the first job appended for a given date, insert a full-width row (merged A:I):
```
▼ MONDAY, 25 Aug 2026 — 3 new jobs
```
- Weekday name uppercase + full date. Update the count as more jobs are added that day
  across the day's runs (or write the count on the day's final run; a running count is
  fine too).
- All jobs from **all 3 runs of that date** go under this one header, appended in arrival
  order. Do **not** create a second header for the same date.

### Job row (one per new posting)
| Col | Value |
|-----|-------|
| A Date | `25 Aug` |
| B Run (IST) | `09:03` (the run time that discovered it) |
| C Company | `Microsoft` |
| D Role Title | `Software Engineer II` |
| E Location | `Hyderabad` (or `Bengaluru; Noida`) |
| F Exp | `0–2 yrs` / `1 yr` / `not stated` |
| G Fit | `8.5` |
| H Job URL | direct apply link |
| I Status | `New` |

## Dedup (Option A)
- Before appending, read existing rows in the **current** week tab (and the previous week
  tab if present) to build the seen-set of normalized `Job URL`s.
- Normalize URLs before comparing: lowercase host, strip tracking query params
  (`utm_*`, `gh_src`, `src`, etc.) and trailing slashes.
- A posting whose URL is already present is **skipped** (not re-logged, not re-emailed).
- Different URLs = different rows, even for the same company/title.

## Formatting (best-effort)
- Bold the header row (row 2) and the day-header rows.
- If the connector supports cell background color, lightly shade day-header rows (e.g.
  alternate soft colors per day). This is optional — the text separator is what matters.
- Freeze rows 1–2 if supported.
