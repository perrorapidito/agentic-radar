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

On a typical Monday, the radar scans 80 companies, finds 22 with open PM or Senior PM roles, narrows it down to 11 that are viable from Spain, validates 7 of those are still actually live, and shows you the top 4 ranked:

```
## Radar — 2026-05-18

| # | Company   | Role                      | Score | The AI's read                                          |
|---|-----------|---------------------------|-------|--------------------------------------------------------|
| 1 | Lumora    | PM — AI Safety Evals      | 89    | ⭐ Builds the artifact you shipped last year.          |
| 2 | DataForge | Senior PM — Ingestion     | 82    | Strong match. Team needs your discovery chops.         |
| 3 | Northwind | PM — Pricing              | 71    | Fintech pivot. Call a recruiter before applying.       |
| 4 | OldCorp   | PM — Internal Tools       | 54    | Discard. No AI footprint, lateral move.                |
```

Each row expands into the full breakdown. The top one looks like this:

```
### Lumora · PM — AI Safety Evaluations  ⭐

Adjusted Score: 89/100  ·  Verdict: Apply NOW
Breakdown: Fit 91 · Financial Health 88 · AI Resilience 86

Differentiating advantages that set you apart:
  · Built an LLM evaluation framework in your previous role —
    the exact artifact this team produces.
  · 8 years in B2B enterprise SaaS — you speak the language of
    Lumora's regulated buyers.
  · Agentic AI in production — covers two job requirements at once.

Real gaps in the fit (not coverable by a cover letter):
  · No formal background in AI safety research (no papers, no
    red-team leadership).
  · C1 English versus a native-speaker team in research-heavy comms.
  · No prior PM experience inside a frontier AI lab.

Questions to validate before applying:
  · Is co-authorship of safety papers expected, or only product
    translation of research findings?
  · Contracting direct from Spain, or via EOR? JD says "Europe"
    but doesn't specify a country.
  · Reports to Head of Safety Research or to VP Product? Defines
    research-led vs. roadmap-led day-to-day.

URL: https://jobs.lumora-ai.com/jobs/91234567
```

You pick which ones belong in your active pipeline. The rest stay in the radar log for next week.

## Running it against the real world (April–May 2026)

After four weeks of using the system, I did a cleanup. Of 21 jobs I had marked as active in my tracker, 13 were dead: the company had closed the position but their website was still showing it. Of the rest, 3 I couldn't even verify because the sites blocked me, and 2 were the same job listed twice.

Almost two-thirds of what I had was well-presented noise.

That's what made me redesign the validation layer from scratch. In the month after the change, not a single dead link.

Without that layer, you spend your week applying to ghosts: jobs that exist on Google but no longer in reality.

## Setup

About 30 minutes the first time. See [SETUP.md](SETUP.md).

## For the curious

- [docs/architecture.md](docs/architecture.md) — why subagents instead of one big loop, token economics, the human-in-the-loop boundary
- [commands/](commands/) — the three slash-command definitions

The orchestration pattern applies beyond job search. Product discovery (ranking 50 user-interview candidates), vendor evaluation, RFPs, grant calls, conference CFPs, real-estate listings: anywhere you have a long-tail of options needing consistent triage with a human-in-the-loop decision step.

## License

MIT.
