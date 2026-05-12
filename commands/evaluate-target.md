# Evaluate Target — Score one opportunity against a user profile

Evaluates a single opportunity (job posting URL, RFP, grant call, etc.) against a user profile and recent history, returning a compact verdict block.

Built for the job-evaluation domain. The scoring weights, location eligibility rules, and output format are adapted to that case — change them for other domains.

## Required inputs

When invoking standalone, the calling context must provide:

- `URL` of the opportunity
- A path to a **user profile summary** file (`./profile/user-profile.md`)
- A path to a **history file** of prior evaluations (`./pipeline/history.md`) — used to reuse company fichas

When invoked from the `/radar` orchestrator, the orchestrator passes these in the subagent prompt.

## Mandatory first actions

In parallel:
- Load the user profile summary from the configured path
- Grep the history file for the company name (to reuse the company ficha if it exists)

If no URL is supplied: *"Profile and history loaded. Paste the JD/URL."*

## Flow

### Step 0 — Location eligibility (FILTER)

Hard rule:
- If the JD says `US only / Remote United States / must be based in [non-eligible city] / no visa sponsorship` → **STOP**. Register as `❌ Discarded — location` and skip scoring.
- If it says `Remote worldwide / EMEA / Europe / Spain / UTC±2` → eligible.
- If it says only `Remote` with no region → continue, flag `LOCATION: ⚠️` and add to *Questions to validate*.

### Step 1 — Preliminary fit (BEFORE WebSearch)

Extract opportunity metadata and compute a preliminary fit score (0–100) by crossing JD keywords and seniority against the user profile. **If preliminary < 50 → discard without WebSearch** (register as `❌ Below threshold`, skip company ficha).

Metadata format:
```
COMPANY · ROLE · MODEL · SENIORITY · SECTOR · LOCATION · URL
```

### Step 2 — Company ficha (only if preliminary ≥ 50)

**Reuse before searching**: if the company already appears in history, reuse its ficha and run a single minimal search (`"[company] layoffs funding [current year]"`) to detect changes since last ficha.

**If new company**: one WebSearch — `"[company] employees funding crunchbase [year]"`. Second search ONLY if runway/funding is missing AND the company is not a known publicly-traded entity.

Compact output:
- If Health = 🟢: one-liner → `[company] · [headcount] · [model] · 🟢 [one sentence]`
- If Health = 🟡/🔴: full ficha (founded, HQ, headcount, model, funding, customers, layoffs)

Score directly 0–100 across three axes:
- Financial stability (runway / revenue / funding rounds)
- Professional attractiveness (CV-builder, impact, culture)
- Job security (Glassdoor, restructurings, management churn)

→ **Health = average of the three** (0–100). No public data → 60.

**Health verdict (unified, 2–3 sentences)**: can the user reasonably land here with 12–24 month stability? Surface the dominant risk (runway, funder dependency, recurring layoffs, management churn, sub-market comp) and whether it's outweighed by upside. Honest even if uncomfortable.

### Step 3 — Full Fit Score and AI Resilience

**Fit Score (0–100)** — one line per dimension + ONE paragraph of justification:

```
Keywords X/30 · Exp X/25 · Achievements X/20 · Soft X/15 · Certs X/10 → TOTAL/100
```

Weights: technical keywords 30 · years+company-type 25 · quantitative achievements 20 · soft+culture 15 · certs 10. Calibrate against recent history entries.

**AI Resilience (0–100, no WebSearch)** — score directly:
- Density of non-automatable tasks (40%): strategy/cross-functional/ambiguity vs reporting/scheduling
- Decision-chain position (35%): Lead/Director with autonomy vs executor
- Sector AI-adoption speed (25%): regulated/physical vs pure technical SaaS

Rating: 🟢 ≥75 · 🟡 45–74 · 🔴 <45.

**AI Resilience verdict (unified, 2–3 sentences)**: does this role *build* or *consume* the user's profile over 3–5 years? Does it move them closer to the strategic decision locus?

### Step 4 — Adjusted Score and verdict

```
Adjusted Score = Fit × 0.60 + Health × 0.25 + Resilience × 0.15
```

- ≥85 → **Apply NOW** ⭐
- 75–84 → **Apply**
- 65–74 → **Decide** (what to validate first)
- <65 → **Discard**

If Resilience 🔴: append *"⚠️ High automation exposure in 2–3 years."*

Always close with (max 3 each):
- **Differentiating advantages** for this role
- **Real gaps** not coverable by a cover letter
- **Questions to validate** before applying

### Step 5 — Update pipeline

Check for duplicates with `Grep "[company]"` on the pipeline file (don't reload the whole file). If it exists, ask whether to update or add as a distinct role. If not, insert a new row at the top of the table replicating the exact format of existing entries.

**Skip this step when invoked from `/radar`** — the orchestrator handles pipeline writes only after explicit user confirmation.

## Principles

- **Never fabricate** company data — if no public info, mark with `❓` and Health = 60.
- **Don't inflate the score** to justify applying.
- **Tone**: senior advisor counseling a peer — direct, no clichés.
