# Hustling_n_Scraping — SDE job scout

Automated SDE job scout for India. A **Claude Agent Skill** (`.claude/skills/job-scout`)
run by a **Claude Routine** on cloud, 3× daily. Each run uses the **Firecrawl MCP
connector** to scrape target companies' career portals, filters for entry-level
software-developer roles (0–2 yrs, on-site India), de-duplicates against a Google
Sheet, logs new matches, and emails a digest.

## How it works

```
Claude Routine (3×/day, IST)
      │  clones this repo, loads the job-scout skill
      ▼
Firecrawl MCP ──► search / map / scrape / extract each company's careers site
      ▼
classify + filter (SDE, 0–2 yrs, on-site India) ──► rank (rule-based fit)
      ▼
Google Sheets connector ──► dedup vs existing rows ──► append new jobs (weekly tab)
      ▼
Gmail connector ──► email digest of new jobs
```

## Repo layout

| Path | Purpose |
| --- | --- |
| `.claude/skills/job-scout/SKILL.md` | Entry-point skill: end-to-end workflow |
| `.claude/skills/job-scout/references/` | On-demand detail (ATS patterns, rubric, formats) |
| `config/profile.md` | Search criteria, notify email, sheet URL, week anchor |
| `config/companies.yaml` | Target companies → ATS type + careers URL |
| `routine-prompt.md` | Paste-in prompt for `claude.ai/code/routines` |
| `.mcp.json` | Project-scope Firecrawl MCP declaration (optional) |
| `docs/` | Reference docs (design plan, Firecrawl reference) — not used at runtime |

## One-time setup

1. **Connectors** (add at `claude.ai/customize/connectors`, all on the same Google account
   that owns the state sheet):
   - **Firecrawl** — remote MCP `https://mcp.firecrawl.dev/v2/mcp` (needs a Firecrawl API key)
   - **Gmail** — sends the digest
   - **Google Sheets** — reads/writes the state + log sheet
2. **GitHub**: push this repo; install the Claude GitHub app on it (or run `/web-setup`).
3. **Plan**: Claude Pro/Max/Team/Enterprise with Claude Code on the web enabled.
4. **State sheet**: the pinned Google Sheet in `config/profile.md` must exist and be
   accessible by the Google account behind the Sheets connector.
5. **Routine**: at `claude.ai/code/routines` → New routine → select this repo → include
   the Firecrawl + Gmail + Google Sheets connectors → paste `routine-prompt.md` → add a
   Schedule trigger (3× daily, IST times of your choice) → Create.

Test with **Run now**, then read the session transcript to confirm the flow.

## Notes

- State lives in Google Sheets, **not** the repo: a routine run re-clones the default
  branch each time and can't persist files back for the next run.
- Adding a company = one line in `config/companies.yaml`. If its ATS isn't covered by an
  `ats-*.md` reference, the skill falls back to `ats-generic.md` (Firecrawl search/map).
