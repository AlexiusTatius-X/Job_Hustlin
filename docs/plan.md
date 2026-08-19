# Plan: Job-Hunting Agent Skill via Firecrawl + Claude Routine

Build a Claude **Agent Skill** (`SKILL.md` + `references/`) in a GitHub repo, run by a scheduled **Claude Routine** on cloud. Each run clones the repo, uses the **Firecrawl MCP connector** to scrape major companies' career portals, filters/ranks SDE roles against fixed criteria, **emails a digest**, and **logs to a Google Sheet**. The routine orchestrates; Firecrawl does the web work.

## The "many websites" problem — solved by ATS patterns, not per-company code
Most MNCs run on a handful of applicant-tracking systems: Workday (`myworkdayjobs.com` — Nvidia + many), Greenhouse, Lever, Ashby, SmartRecruiters, iCIMS, Taleo, Eightfold, plus custom ones (Microsoft, Google, Apple, Amazon). One reusable extraction pattern per ATS (one reference file each), so adding a company is just a config line. Odd/non-standard titles (e.g. Apple) are classified by JD content, not title text.

## Confirmed facts (researched)
- **Routines**: Pro/Max/Team/Enterprise with Claude Code on the web. Created at `claude.ai/code/routines`.
- Routine = prompt + GitHub repo(s) + connectors + triggers (schedule/API/GitHub).
- Runs autonomously as a full Claude Code cloud session; can run shell, use skills **committed to the cloned repo**, and call MCP connectors.
- Schedule min interval = 1 hour; daily run cap applies; times convert local → UTC.
- **Key constraint (read-only per run)**: each run re-clones the *default branch* and pushes only to `claude/`-prefixed branches → files written during a run are **NOT visible next run**. Repo = static inputs only; mutable state lives in Google Sheets.
- Firecrawl MCP connector traffic routes via Anthropic's servers → **no network-allowlist edits** needed.
- Firecrawl remote MCP: `https://mcp.firecrawl.dev/v2/mcp` — added at `claude.ai/customize/connectors`.
- Firecrawl tools: `search`, `map`, `scrape`, `extract` (structured), `interact`, `monitor`.

## How the skill loads (progressive disclosure)
- Skill = a folder whose entry point is `SKILL.md`. Frontmatter (`name`+`description`) always loaded; body loaded on trigger (keep < ~500 lines); `references/*.md` read on demand only.
- In an autonomous routine, the prompt should **explicitly** tell Claude to load and follow the `job-scout` skill.

## Architecture decisions
- **Primary orchestrator = Claude Routine** (3×/day). Firecrawl MCP does the web work.
- Do **not** rely on Firecrawl `monitor` as primary — it can't reason about *fit* or aggregate across companies.
- Solve "many websites" via **ATS-platform recognition** (one reference file per ATS).
- Connectors are per-routine and grant ALL tools (incl. writes) with no per-run approval → include only: Firecrawl, Gmail, Google Sheets.

## Job criteria (FINAL)
- Role: **SDE / SDE-1 / SWE** + equivalents (frontend / backend / full-stack). Classify by JD content (handles odd titles).
- Experience: **strictly 0–2 years**. Reject anything requiring > 2 yrs.
- **Exclude**: internships (all), ML/AI engineer, data scientist, applied scientist, research, AI/ML-software roles.
- Location: **on-site, INDIA only**. Priority cities: Bangalore, Hyderabad, Noida, Gurgaon. Reject remote / non-India.

## Fit-scoring (NO résumé — user opted out)
Rule-based relevance score, not a personal-résumé match:
- Title-match strength (exact SDE > equivalent > classified-in-by-JD)
- Experience fit (clean 0–2y > unstated-but-junior)
- Location priority (BLR/HYD/NOIDA/GGN > other India)
- Freshness (recency of posting)

## Repo structure (repo root = `Hustling_n_Scraping`)
```
.  (repo root)
├── .claude/skills/job-scout/
│   ├── SKILL.md                 # root: orchestration flow, links to references
│   └── references/
│       ├── ats-workday.md
│       ├── ats-greenhouse.md
│       ├── ats-lever.md
│       ├── ats-generic.md       # fallback via Firecrawl search/map
│       ├── matching-rubric.md   # role rules + rule-based fit score
│       ├── sheet-format.md      # weekly tabs, day headers, columns, dedup
│       └── email-format.md      # digest layout
├── config/
│   ├── profile.md               # criteria + notify_email + sheet URL + week anchor
│   └── companies.yaml           # company → ATS type + careers URL
├── docs/                        # reference only (plan.md, firecrawl-reference.md)
├── routine-prompt.md            # paste-in prompt for claude.ai/code/routines
├── .mcp.json                    # project-scope Firecrawl MCP (optional)
└── README.md
```
Static inputs only (read-only per run). Mutable state → Google Sheets.

## Delivery — Email (CONFIRMED)
- Digest emailed to **oniidaddy881@gmail.com** via the **Gmail connector**.

## State / dedup + log — Google Sheets (CONFIRMED)
- Pinned spreadsheet: `https://docs.google.com/spreadsheets/d/1PsxcTYjLEbmK2MfRKmQBh9l9BZhhJZEC5ARx8w9l9TA/edit`
- **One tab per week**, auto-created each Monday. Week = **Mon 00:00 → Sun 23:59 IST**. Each run computes "this week's Monday"; if the week-tab is missing, create it, then append (deterministic, no mutable state).
- Partial first week (first run mid-week) = **counts as Week 1 (partial)**; next Monday = Week 2.
- Tab naming = **`Week 01 · 25–31 Aug`** (sequential `NN = floor((thisMonday − anchorMonday)/7)+1`; `anchorMonday` pinned in `config/profile.md`).
- Inside a tab: a **day-header separator row** before each day's block (e.g. `▼ MONDAY, 25 Aug 2026 — N new jobs`); all 3 daily runs' finds grouped under that day.
- Columns: `Date, Run(IST), Company, Role Title, Location, Exp, Fit, Job URL, Job ID, Status`.
- **Dedup = Option A**: same posting (same job_id/url) logged **once** at first-seen; later runs skip it. Distinct postings = distinct rows.
- Per-day color shading = nice-to-have (depends on connector); day-header row is the reliable separator.

## Cadence — 3×/day (CONFIRMED)
- IST. Exact run times set manually by user in the routine UI. (Routine min interval = 1h; daily cap applies.)

## Company list — ~45 (v1 approved)
Compiled in `config/companies.yaml`: big-tech, product unicorns, quant/HFT, fintech/banks. ATS type per company; `# verify` entries confirmed on first build run via Firecrawl map.

## Prerequisites (user, at build time)
1. Firecrawl API key + add Firecrawl MCP connector on claude.ai.
2. Add **Gmail** + **Google Sheets** connectors (same Google account that owns the sheet).
3. GitHub repo + install Claude GitHub app / `/web-setup`.
4. Confirm Pro/Max/Team/Enterprise + Claude Code on the web enabled.

## Steps
1. **Prereqs** (above).
2. **Config** — `config/profile.md` + `config/companies.yaml`.
3. **References** — `references/ats-*.md`, `matching-rubric.md`, `sheet-format.md`, `email-format.md`.
4. **The skill** — `.claude/skills/job-scout/SKILL.md`: `search → map → scrape → extract → classify/filter → dedup(vs Sheet) → rank → append(Sheet) → email` flow.
5. **Create routine** — at `claude.ai/code/routines`: select repo, include Firecrawl + Gmail + Sheets connectors, paste `routine-prompt.md`, 3×/day IST schedule (set manually).
6. **Test & iterate** — "Run now"; review transcript; refine.

## Verification
1. "Run now" → transcript shows Firecrawl tools called, jobs extracted/classified, Sheet appended, email sent.
2. Dedup: second run same day yields no duplicate postings in the Sheet/email.
3. Week rollover: a Monday run creates a new `Week NN` tab; day-header separators render correctly.
4. Spot-check 3–5 returned jobs against live career pages for accuracy + freshness (SDE, 0–2y, India on-site).
5. Confirm scheduled triggers fire and deliver over a full day/week.

## Deferred / non-blocking
- Résumé/skills summary: **dropped** — skill functions without it (rule-based fit only).
