# Agentic Radar

A weekly AI radar for job-hunting at scale. Built with Claude Code.

Built by [Francisco de la Cortina](https://www.linkedin.com/in/fdelacortina/). Eight years in Product Management, the last four shipping agentic AI in enterprise SaaS. 

> 🌐 **English** · [Español](README.es.md)

---

## The pain

You've curated 80 target companies. Every Monday: open 80 careers pages, cross-reference noisy LinkedIn alerts, lose an hour on a "remote" role that's actually US-only, decide on gut feel because there's no time to research the company's funding, team, runway, AI strategy. Three months in, you're applying to whatever's loudest. That's burnout management, not a hiring strategy.

## What it does

A cron runs three commands every Monday: scan the inventory via job-board APIs, filter to PM/Senior PM with a viable location, evaluate each survivor against your profile in an isolated AI subagent. Monday morning, you find a ranked table with verdicts, real gaps, and the questions to validate before applying. You decide in five minutes.

## What you actually get

```
## Radar — 2026-05-18

| # | Company   | Role                      | Score | The AI's read                                          |
|---|-----------|---------------------------|-------|--------------------------------------------------------|
| 1 | Lumora    | PM — AI Safety Evals      | 89    | ⭐ Builds the artifact you shipped last year.          |
| 2 | DataForge | Senior PM — Ingestion     | 82    | Strong match. Team needs your discovery chops.         |
| 3 | Northwind | PM — Pricing              | 71    | Fintech pivot. Call a recruiter before applying.       |
| 4 | OldCorp   | PM — Internal Tools       | 54    | Discard. No AI footprint, lateral move.                |
```

Below the table, each row expands into the full breakdown: fit / health / resilience sub-scores, top advantages, real gaps, and the three questions to validate before applying. You pick which ones belong in your active pipeline. The rest stay in the radar log for next week's recheck.

## A real run (April–May 2026)

After four weeks of using this I audited my own tracker. The numbers that made me redesign the whole scan layer:

- **21 URLs marked "active"** from prior weeks
- **13 were dead** (62%): Greenhouse `?error=true` redirects, Ashby rerouting closed jobs to board root, Lever 404s
- **3 unverifiable** behind anti-bot walls (Revolut, Cloudbeds careers JS)
- **5 actually valid**, of which 2 were duplicates of the same role

After moving from HTML scraping to JSON APIs and adding HEAD validation as a gate: 11/11 URLs reported in the next scan returned HTTP 200 with non-error effective URL. Zero false positives in the month after.

That 62% → 0% delta is why the validation layer is non-negotiable. Without it, you're applying to ghosts.

## What this doesn't do (yet)

A few things I haven't solved. Listed in order of how much they bug me.

- **Workday tenants block the CXS API after the first request.** Several large enterprise targets in my inventory end up in a partial blind spot. WebFetch fallback works but it's flaky and the URLs need manual verification.
- **LinkedIn-only postings are invisible.** No public API. If a company posts only there, the radar misses them entirely.
- **Title filtering is keyword-based, not semantic.** It catches "Principal PM" correctly, but a creative title like "Product Strategy Lead" would slip through. Embedding-based filtering is on my list.
- **No JD change detection between scans.** If a posting silently tightens requirements or changes location, I won't notice until I open it.
- **Salary signal is missing.** Most JDs don't publish bands. Enriching via Levels.fyi or similar is the obvious next step.

## Setup

About 30 minutes the first time. See [SETUP.md](SETUP.md).

## For the curious

- [docs/architecture.md](docs/architecture.md) — why subagents instead of one big loop, token economics, the human-in-the-loop boundary
- [docs/lessons-learned.md](docs/lessons-learned.md) — 8 concrete bugs I caught while building this (anti-bot patterns, JS-rendered pages, hallucinated URLs, location parsing edge cases)
- [commands/](commands/) — the three slash-command definitions

The orchestration pattern applies beyond job search. Vendor evaluation, RFPs, grant calls, conference CFPs, real-estate listings: anywhere you have a long-tail of options needing consistent triage with a human-in-the-loop decision step.

## License

MIT.
