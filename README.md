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

| # | Company    | Role                              | Score  | The AI's read                                                        |
|---|------------|-----------------------------------|--------|----------------------------------------------------------------------|
| 1 | Lumora AI  | PM — AI Safety Evaluations        | 89/100 | ⭐ Apply NOW — "Builds the exact artifact you already shipped last year." |
| 2 | DataForge  | Senior PM — Streaming Ingestion   | 82/100 | Apply — "Strong technical match; the team needs your customer-discovery chops."  |
| 3 | Northwind  | PM — Pricing & Monetization       | 71/100 | Decide — "Fintech pivot; worth a recruiter call before investing in a cover letter." |
| 4 | Kraken DE  | PM — German Energy Market         | 68/100 | Decide — "Spain-adjacent via EOR, but the JD reads native-German for discovery."   |
| 5 | OldCorp    | PM — Internal Tools               | 54/100 | Discard — "No AI footprint, no public-facing impact — strictly lateral."          |
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

## Set this up yourself — no coding needed

This is built for **[Claude Code](https://claude.com/claude-code)**, a desktop app from Anthropic that lets you chat with Claude *and* have it do real things on your computer — read files, edit them, run scheduled tasks. Think of it as "Claude with hands".

If you can write a job-search spreadsheet, you can set this up. **Estimated time: 30–45 minutes the first time. After that, it runs itself every Monday.**

### Step 1 — Install Claude Code (5 min)

Download from [claude.com/claude-code](https://claude.com/claude-code) and sign in. It's a normal Mac/Windows app.

### Step 2 — Add the three "skills" (5 min)

A "slash command" in Claude Code is a pre-written instruction that you trigger by typing `/something` in chat. This repo gives you three of them:

1. Open Claude Code.
2. In the chat, type: `/init` — Claude will create the folder structure for you, including a `~/.claude/commands/` directory.
3. Download the three files inside the [`commands/`](commands/) folder of this repo (`scan-targets.md`, `evaluate-target.md`, `radar.md`) and drop them into your `~/.claude/commands/` folder.
4. Restart Claude Code. Type `/` in the chat — you should now see `/scan-targets`, `/evaluate-target`, and `/radar` in the list.

That's it. No code written. The "skills" are just markdown files describing what Claude should do; Claude does the actual work.

### Step 3 — Build your target list (10 min)

Make a simple list of the companies you want the radar to check every week. The format is shown in [`examples/targets-inventory.example.md`](examples/targets-inventory.example.md) — a markdown table you can edit in any text editor (or VS Code, or Notion, or even on GitHub directly).

Each row needs only **company name + careers page URL**. The radar will figure out the rest on its first run.

Save it somewhere private (your own folder, a private Google Doc, a private GitHub repo). Don't put it in the public version of this repo — it's in `.gitignore` for that reason.

### Step 4 — Tell Claude what kind of role you want (5 min)

This is the only "configuration" step. Open `commands/scan-targets.md` in Claude Code and ask Claude in chat:

> *"In the file `commands/scan-targets.md`, change the title filter to also accept Principal PM"*

Or:

> *"Change the location filter so it accepts Berlin and Amsterdam as valid locations"*

You don't have to edit the file by hand — just tell Claude what you want and it edits the file for you. (This is the actual workflow people use Claude Code for.)

### Step 5 — Run it once (2 min)

In Claude Code chat, type: `/radar`

The first run takes longer because the system has to figure out which job-board platform each company uses. After it finishes, you'll see the ranked table — exactly like the one in the example above.

### Step 6 — Schedule it weekly (1 min)

Type in chat: `/schedule`

When asked, give it:
- **Name**: `radar-weekly`
- **Cron**: `0 8 * * 1` (means "every Monday at 8:00 AM")
- **Command**: `/radar`

Now every Monday morning you'll find the radar's output waiting for you. You spend 5 minutes deciding what to pursue. Done.

### What if I get stuck?

Ask Claude directly in the chat. Examples that work:

> *"The scan returned 0 hits for Company X but I know they're hiring — what's wrong?"*

> *"Re-run the radar but only on these 3 companies: Lumora, DataForge, Northwind"*

> *"Add a new company to my target list: Atomic AI, careers page is https://atomic.ai/jobs"*

The skills are written so Claude can fix, extend, and operate them through normal conversation. You don't need to learn YAML, Python, or any framework — just talk to it.

## For the curious

If you want the technical depth:

- [docs/architecture.md](docs/architecture.md) — why subagents instead of one big loop, the token economics, the human-in-the-loop boundary
- [docs/lessons-learned.md](docs/lessons-learned.md) — the 8 specific bugs I caught while building this: anti-bot patterns, JS-rendered pages, hallucinated URLs, location parsing edge cases
- [commands/](commands/) — the three slash-command definitions, fully annotated

The pattern generalizes beyond job-hunting. Anywhere you have a long tail of options (vendors, RFPs, grant calls, conference CFPs, real-estate listings) and need consistent, evidence-based triage with a human-in-the-loop decision step, this same architecture applies. Swap the title filter, swap the scoring weights, change the inventory source — the orchestrator stays the same.

## License

MIT. Fork it. Adapt it. Make your own ritual a little less terrible.
