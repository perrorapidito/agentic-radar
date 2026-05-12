# Agentic Radar

> A weekly AI radar that scans your target companies, finds the right openings, and tells you what's worth applying to — built with Claude Code slash commands.

## The problem this solves

You've curated a list of 80 companies you'd love to work at. Every Monday, the ritual:

- Open 80 careers pages. Most have no Product Manager openings. Some redirect to dead links. A few have 47 jobs but you only care about 1.
- Check LinkedIn alerts. They show you the same opening 6 times across different agencies.
- Lose an hour to a single posting that turns out to be US-only with no Spain remote.
- By the time you finish the list, the first ones you checked already have new postings.
- Decide whether to apply based on gut feel, because there's no time to actually research the company's funding, team, runway, AI strategy.

**Three months in, you're applying to whatever's loudest.** That's not a hiring strategy, that's burnout management.

This radar is what I built to escape that loop.

## What it actually does

Every Monday at 8am, it:

1. **Scans** your target companies via their job-board APIs (Greenhouse, Lever, Ashby, Workable, Personio). Validates every URL before reporting it — no dead links.
2. **Filters** to titles you actually want (Product Manager / Senior PM, not Principal/Staff/Director) and locations you can actually take (Spain, EMEA-remote, worldwide — never US-only).
3. **Evaluates** each surviving opportunity in parallel — an isolated AI subagent looks up the company's funding, headcount, layoffs, and culture, then scores the role against your profile across three dimensions: fit, financial health, and AI resilience.
4. **Hands you a ranked list** with verdicts (`Apply NOW`, `Decide`, `Discard`) and the three real gaps + three real questions to ask before applying.

You spend 5 minutes Monday morning deciding which to pursue. The radar did the 4 hours of legwork while you slept.

## What it looks like (fictional example)

After a Monday run, you open Claude Code to this in chat:

```
## Radar — Evaluation 2026-05-18

| # | Company    | Role                              | Score  | Verdict       |
|---|------------|-----------------------------------|--------|---------------|
| 1 | Lumora AI  | PM — AI Safety Evaluations        | 89/100 | ⭐ Apply NOW  |
| 2 | DataForge  | Senior PM — Streaming Ingestion   | 82/100 | Apply         |
| 3 | Northwind  | PM — Pricing & Monetization       | 71/100 | Decide        |
| 4 | Kraken DE  | PM — German Energy Market         | 68/100 | Decide        |
| 5 | OldCorp    | PM — Internal Tools               | 54/100 | Discard       |
```

And below it, the detailed block for each. The top hit looks like this:

```
### Lumora AI · PM — AI Safety Evaluations  ⭐
- Adjusted Score: 89/100
- Verdict: Apply NOW
- Breakdown: Fit 91 · Health 88 · Resilience 86

Differentiating advantages (top 3):
- Built an LLM evaluation framework in your last role —
  exact artifact this team produces (red-teaming, eval metrics, HITL loops).
- 8 years B2B enterprise SaaS — speaks the language of Lumora's buyers
  (regulated industries, sovereign AI nicho).
- Multi-cloud + agentic AI in production — covers two job requirements at once.

Real gaps (top 3):
- No formal AI safety research background (no papers, no red-team leadership).
- C1 English vs. native-speaker team in research-heavy comms.
- No previous PM experience inside a frontier AI lab (you've applied LLMs,
  not trained foundation models).

Questions to validate before applying:
- Is co-authorship of safety papers expected, or only product translation
  of research findings?
- Spain contracting direct or via EOR? (Europe listed but country not confirmed.)
- Reports to Head of Safety Research or VP Product?
  → defines research-led vs. roadmap-led day-to-day.

URL: https://jobs.lumora-ai.com/jobs/91234567  (HEAD-validated 2026-05-18 08:14)
```

You read four of these, decide `1, 3` belong in your active pipeline, and tell the radar. It writes those two to your tracker; the rest stay in the radar log for next week's recheck.

**That's it. That's the workflow.**

## Why it works

Three design decisions, each of which I had to learn the hard way:

### 1. Talk to the APIs, not the websites

Every major hiring platform (Greenhouse, Lever, Ashby, Workable, Personio) exposes a JSON feed of open positions. Reading the website instead is how you end up with hallucinated URLs that look right but don't exist, and dead links cached by Google three weeks ago.

### 2. Validate every URL before showing it to a human

A simple HEAD request, but done strictly. Greenhouse silently redirects deleted jobs to `?error=true`. Ashby reroutes closed postings to the board root. Without validation, ~60% of "active" listings reported to you are actually dead within four weeks.

### 3. Use subagents for the deep work, keep the human in the main loop

Each opportunity evaluation involves 15K tokens of company research, JD parsing, and scoring. Doing 7 of those inline destroys the AI's reasoning quality for everything that comes after. Sending each to an isolated AI subagent that returns only a 400-word verdict block keeps your main context clean — and the subagents run 3 at a time in parallel, so the whole batch takes ~2 minutes.

The human gets the rankings and decides what enters their pipeline. The AI never auto-applies, never auto-promotes. That separation is what makes the system trustworthy enough to run weekly without supervision.

## Try it on your own search

This is built for [Claude Code](https://claude.com/claude-code). To adopt it:

1. **Drop the three files in `commands/` into your `~/.claude/commands/`.** They become `/scan-targets`, `/evaluate-target`, and `/radar`.
2. **Build your inventory.** Copy `examples/targets-inventory.example.md` and replace the placeholder rows with your real target companies + careers URLs. Don't commit this file — it's in `.gitignore` by default.
3. **Tune the filters** in `commands/scan-targets.md`:
   - `TITLE_BLOCKERS` — which titles to exclude (Principal, Staff, Director, etc.)
   - Location lists (`SPAIN_STRONG`, `EMEA_PLACES`, `US_ONLY_PLACES`) — where you can actually work
4. **Run `/radar`.** First run takes longer because it discovers each company's ATS slug; subsequent runs are fast.
5. **Schedule it** via Claude Code's `/schedule` once you trust the output. Recommended: `0 8 * * 1` (Mondays 8am, your timezone).

## For the curious

If you want the technical depth:

- [docs/architecture.md](docs/architecture.md) — why subagents instead of one big loop, the token economics, the human-in-the-loop boundary
- [docs/lessons-learned.md](docs/lessons-learned.md) — the 8 specific bugs I caught while building this: anti-bot patterns, JS-rendered pages, hallucinated URLs, location parsing edge cases
- [commands/](commands/) — the three slash-command definitions, fully annotated

The pattern generalizes beyond job-hunting. Anywhere you have a long tail of options (vendors, RFPs, grant calls, conference CFPs, real-estate listings) and need consistent, evidence-based triage with a human-in-the-loop decision step, this same architecture applies. Swap the title filter, swap the scoring weights, change the inventory source — the orchestrator stays the same.

## License

MIT. Fork it. Adapt it. Make your own ritual a little less terrible.
