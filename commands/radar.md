# Radar — Scan + Evaluation orchestrator

Chains `/scan-targets` and `/evaluate-target` to produce, in one execution, a ranked decision-ready table.

## Flow

```
Step A → run /scan-targets (no args, or pass through only:<X>)
Step B → filter hits to verdict ∈ {OK, EMEA_CTRY} — drop AMBIG (user verifies)
Step C → for each surviving hit, spawn a general-purpose subagent running /evaluate-target
          → batches of 3 in parallel, wait for full batch before next
Step D → aggregate verdicts into a ranked table by Adjusted Score desc
Step E → ask user which positions to add to the pipeline
```

## Step A — Scan

Run the full `scan-targets` flow. At the end, an in-memory structured list of hits with fields `(source, company, role, location, verdict, url)`.

Accepts the same args as `scan-targets`: by default the whole inventory; `only:<tier|company>` to narrow.

## Step B — Filter for evaluation

Keep only hits with `verdict ∈ {OK, EMEA_CTRY}`. `AMBIG` hits (bare-remote with no region) appear in the scan report but are NOT auto-evaluated — the user verifies them manually because the location is uncertain.

Communicate before Step C:
> "Scan complete: N total hits (OK: X · EMEA_CTRY: Y · AMBIG: Z). Going to evaluate the X+Y survivors in batches of 3 via subagents."

## Step C — Spawn subagents in batches of 3

For each hit to evaluate, launch a `general-purpose` subagent with this prompt (substitute the variables in `{...}`):

```
You are evaluating one opportunity against a user profile.

FIXED INPUT:
- URL: {URL}
- Company: {COMPANY}
- Role: {ROLE}
- Detected location: {LOCATION}
- Location verdict: {VERDICT} (OK = direct match; EMEA_CTRY = viable via contractor)

INSTRUCTIONS:
1. Read the full logic in ./commands/evaluate-target.md and follow it.
2. Load user profile from ./profile/user-profile.md
3. Load company history: grep "{COMPANY}" against ./pipeline/history.md
4. Fetch the JD from {URL} (WebFetch or Read).
5. Execute STEPS 0-4: location eligibility, preliminary score, company ficha, full fit score + AI resilience, adjusted score.
6. SKIP STEP 5 — DO NOT modify the pipeline file. The orchestrator decides later after user confirmation.

OUTPUT — return EXACTLY this block, nothing extra, max 400 words:

### {COMPANY} · {ROLE}
- **Adjusted Score:** XX/100
- **Verdict:** [Apply NOW ⭐ | Apply | Decide | Discard]
- **Breakdown:** Fit XX/100 · Health XX/100 · Resilience XX/100
- **Differentiating advantages (max 3):** ...
- **Real gaps (max 3):** ...
- **Questions to validate (max 3):** ...
- **URL:** {URL}

If AI Resilience is 🔴, append "⚠️ High automation exposure in 2–3 years." to the verdict.
```

**Batch execution**:
- Batch N = hits[N*3 : (N+1)*3]
- Launch the 3 subagents in a single call with multiple Agent blocks (native parallelism)
- Wait for all 3 to return before launching the next batch
- For 7 hits → 3 batches (3+3+1)

## Step D — Aggregate and rank

Collect the subagent blocks and build a table ordered by Adjusted Score desc:

```
## Radar — Evaluation {date}

| # | Company | Role | Adjusted Score | Verdict | URL |
|---|---|---|---|---|---|
| 1 | X | Senior PM Y | 87/100 | ⭐ Apply NOW | url |
| 2 | ... | ... | ... | ... | ... |
```

Below the table, insert the complete detail blocks in ranking order (don't summarize — the user needs them to decide).

## Step E — Pipeline confirmation

Ask the user:

> "I evaluated N positions. The ⭐ Apply NOW ones are [list]. Which do you want me to add to the pipeline file? Accept: 'all', 'only apply-now', 'none', or specific numbers (e.g. 1,3,5)."

Based on the response, execute STEP 5 of `evaluate-target` (insert into pipeline file respecting existing format) **only for confirmed positions**. The rest stay in `active-opportunities.md` only (already written by Step A).

## Args

- **No args** → scans the full inventory + evaluates OK/EMEA_CTRY. Default.
- `/radar only:<tier|company>` — restrict to a subset.
- `/radar skip-scan` — use hits already in `active-opportunities.md` without re-scanning (useful to re-evaluate after profile/filter changes, without spending discovery tokens).

## Critical notes

- **Subagents return compact verdicts only**, never JDs or WebSearches. That's what keeps the main context clean even with 20+ hits.
- **Batches of 3** avoid WebSearch rate-limit spikes.
- **AMBIG (bare-remote) is never auto-evaluated** — location uncertainty makes the eligibility score unreliable. The user verifies manually and, if interested, runs `/evaluate-target <url>` ad-hoc.
- **If a subagent fails** (timeout, WebFetch 403, etc.), include it in the table with `❓ Error — re-evaluate manually` instead of dropping silently.
- **Scrupulous close**: only write to the pipeline file with explicit user confirmation in Step E. Never auto-add.
