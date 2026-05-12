# Architecture

> 🌐 **English** · [Español](architecture.es.md)

## The problem

You have a large, slowly-changing universe of options (companies, vendors, RFPs, grants, user-interview candidates for product discovery, listings) and you need to:

1. Discover which options have **active opportunities** right now.
2. Filter them by **objective constraints** (location, role title, deadline).
3. Score the survivors against a **personal profile** so you can decide what to engage with.

Doing this by hand against 80+ targets weekly burns 4–6 hours and yields stale data — by the time you finish scrolling careers pages, the first ones you checked have changed.

Doing it with a single LLM "agent" call doesn't scale either: each opportunity evaluation is 5–15K tokens (company research + JD parsing + scoring), and 7 evaluations crammed into one context degrades reasoning quality and crashes against the context window.

## The pattern

Three layers, each with a clear contract:

### Layer 1 — Discovery (`scan-targets`)

A fan-out that, for each target in the inventory, queries the most authoritative source available — typically the ATS JSON API of the company's careers page. Returns a structured list of opportunities with verbatim URLs. Validates every URL with a HEAD request before reporting.

Key property: **the discovery layer never invents data**. URLs come from JSON fields copied verbatim. Locations come from structured fields. The only AI inference happens at title classification and location categorization, both bounded to keyword-list matching.

### Layer 2 — Filter (inside `scan-targets`)

Two filters stacked:

1. **Title filter** — bounded inclusion (`product manager`) plus an explicit block-list (`principal`, `staff`, `lead`, `group`, `marketing`, `technical program`, `intern`, ...). This is the place to tune what counts as "in scope" for your specific role band.
2. **Location filter** — a verdict in {OK, EMEA_CTRY, AMBIG, NO}, computed by keyword precedence (strong markers → contractor-viable → ambiguous → reject). The verdict drives downstream behavior: OK and EMEA_CTRY survive to evaluation, AMBIG goes to a separate "verify manually" lane, NO is dropped.

### Layer 3 — Evaluation (`evaluate-target`, orchestrated by `radar`)

For each survivor of the filter, run a deep evaluation in a **separated subagent context**:

- Load the user profile
- Reuse any prior ficha on the company (history file grep)
- Fetch the opportunity content (JD)
- Score against four dimensions: Fit, Health, Resilience, with explicit weights
- Compute Adjusted Score and a verdict band (Apply NOW / Apply / Decide / Discard)
- Return a 400-word block

The orchestrator (`radar`) launches these subagents in **batches of 3 in parallel**, waits for each batch to complete, then continues. The main context only sees the returned 400-word blocks, never the raw research.

## Why subagents and not a single loop

A natural alternative: a single agent that loops `for opportunity in opportunities: evaluate(opportunity)`. This has three problems:

1. **Context window**. After ~5 evaluations of raw JD + WebSearch results, the context is saturated and the model starts losing earlier context.
2. **Reasoning quality**. Even before saturation, irrelevant historical context (the JD of opportunity #1) degrades reasoning about opportunity #5. The model gets distracted by patterns it shouldn't generalize.
3. **Recovery**. If one evaluation fails (timeout, 403, malformed JD), it pollutes the rest. With subagents, failures are isolated; the orchestrator marks one as `❓ Error` and continues.

Batches of 3 are an empirical sweet spot: parallelism without rate-limit pressure on WebSearch and tool APIs.

## Why human-in-the-loop for the pipeline write

Reading is reversible: if the scan misses an opportunity, you'll see it next week. Writing to your active pipeline is **not** reversible: an auto-added entry becomes part of your decision context, and a wrong add silently biases your future judgment.

The pattern explicitly separates the two:
- `scan-targets` writes to `active-opportunities.md` automatically (the radar list, low cost of false positives because you can scan it)
- `radar` writes to the **pipeline file** (the things you're actively pursuing) only after explicit user confirmation

This is the inverse of the "fully autonomous agent" framing common in AI agent demos. It's what makes the system trustworthy enough to schedule weekly without supervision.

## Token economics

Empirical, from a real week's scan:

| Stage | Where | Tokens |
|---|---|---|
| Discovery (scan-targets) | main context | ~5K |
| Per-opportunity deep eval (subagent) | isolated context | ~15K × N |
| Verdict block returned to main | main context | ~400 words × N ≈ ~3K |

For N=7 hits: ~5K + 7×~15K + ~3K = **~113K tokens total**, but only **~8K visible in main context**. The main context stays clean for follow-up reasoning ("what's the best one to apply to first?").

Without subagents: same ~113K tokens but all in one context → degraded reasoning quality, especially in the last 30% of the chain.

## Trade-offs to know

- **Token cost is similar to inline; the win is in reasoning quality, not cost.** Don't promise yourself you'll save tokens by going subagent — that's not where the value is.
- **JSON APIs evolve.** Workable changed its endpoint path between versions. Plan for periodic re-validation of handlers.
- **Anti-bot is the silent assassin.** Workday returns 200 OK then fails on the next request. Greenhouse silently redirects to `?error=true`. Build the validation gate up front, not as an afterthought.
- **Filters are domain-specific.** The title block-list and location keyword lists in this repo are tuned for "Spain-based AI PM hunting in late 2025/early 2026". They will rot. Treat them as configuration, not framework.
- **The pattern doesn't help for one-shot evaluation.** If you have 1 opportunity to evaluate, just run the evaluation inline. The orchestration overhead only pays off at N ≥ 5.
