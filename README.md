# Agentic Radar

> A pattern for evaluating a long-tail of options in parallel with AI subagents — built as Claude Code slash commands. Job-search instantiation included.

## What this is

A three-skill pipeline that orchestrates AI subagents to **discover**, **filter**, and **evaluate** a large universe of options against a personal profile, with the human always in the loop for final decisions.

The concrete example domain is a **job-hunt radar**: scanning ~80 target companies weekly, surfacing Product Manager and Senior PM openings that are location-viable, and producing ranked, scored evaluations of each.

The same pattern applies to: vendor selection, conference talk submissions, M&A pipeline, grant applications, real-estate listings — anywhere you have many options and need consistent, evidence-based triage.

## Architecture

```
┌─────────────────┐   ┌──────────────────┐   ┌──────────────────────────┐
│ /scan-targets   │ → │ filter (location)│ → │ /radar (orchestrator)    │
│ JSON APIs +     │   │ + verdict tag    │   │  ├─ spawn N subagents    │
│ HEAD validation │   │ (OK/EMEA_CTRY/   │   │  │  in batches of 3      │
│                 │   │  AMBIG/NO)       │   │  ├─ each runs /evaluate  │
│                 │   │                  │   │  │  in isolated context  │
│                 │   │                  │   │  └─ aggregate compact    │
│                 │   │                  │   │     verdict per option   │
└─────────────────┘   └──────────────────┘   └──────────────────────────┘
                                                         │
                                              ┌──────────▼──────────┐
                                              │ ranked table → user │
                                              │ confirms pipeline   │
                                              │ adds (never auto)   │
                                              └─────────────────────┘
```

## The three skills

| Skill | Job |
|---|---|
| [`scan-targets.md`](commands/scan-targets.md) | Discovers open opportunities across an inventory of targets. Uses JSON APIs (Greenhouse, Lever, Ashby, Workable, Personio) when possible; falls back to WebFetch for proprietary career sites. Every URL is HEAD-validated before being reported. |
| [`evaluate-target.md`](commands/evaluate-target.md) | Scores a single opportunity against a user profile. Returns a compact block: total score, sub-scores, advantages, gaps, questions to validate. |
| [`radar.md`](commands/radar.md) | Orchestrator. Chains the scan → filters by location verdict → spawns subagents (batches of 3) that each run `evaluate-target` in isolation → aggregates a ranked table → asks the user which to add to the pipeline. |

## Design principles

1. **JSON APIs before scraping**. Every major ATS (Greenhouse, Lever, Ashby, Workable, Personio) exposes a JSON board. Use it. Cleaner data, canonical URLs, no hallucination, ~10× cheaper in tokens than HTML scraping.
2. **Validate every URL before reporting**. A HEAD request gate catches dead links, ATS error redirects (`?error=true`), and stale board fall-throughs. Without this gate, ~60% of URLs in a typical scan are stale within 4 weeks.
3. **Subagents protect the main context**. Each opportunity evaluation can be 5–15K tokens (WebSearch on company + JD parsing + scoring). Running them all in main context exhausts the window. Subagents run in parallel batches of 3, return only a 400-word verdict block.
4. **The human stays in the loop for irreversible state**. The radar produces a ranked table and *asks* before writing to `pipeline.md`. The pattern explicitly refuses automation of "what should I apply to" — that's a human decision, not an inference.
5. **WebSearch only for slug discovery, never for URLs**. Google indexes job boards with a 1–4 week lag. Resolving an opportunity URL via search returns dead links. Search is only used to discover *which ATS* a company uses; URLs always come from the live API.

## Lessons learned

Real bugs caught while building this. Useful if you're implementing something similar:

- **Ashby pages render with JavaScript**. The HTML payload is a skeleton; jobs come from `https://api.ashbyhq.com/posting-api/job-board/{slug}`. WebFetch on the careers page returns nothing useful — go to the API.
- **Ashby `locationName` is often empty**. The real location lives in `workplaceType` + `secondaryLocations[].location` + `isRemote`. A naive filter on `locationName` drops legitimate hits.
- **Workable's `/api/v3/accounts/{slug}/jobs` returns 404**. The working endpoint is `/api/v1/widget/accounts/{slug}` returning `{name, jobs[]}`.
- **Greenhouse redirects deleted jobs to `{slug}?error=true`** (200 OK). HEAD-validate the *effective URL*, not just status code, or you'll think dead jobs are alive.
- **Workday CXS API (`/wday/cxs/{tenant}/{site}/jobs`) is anti-bot aggressive**. First call may succeed; subsequent calls return 400/422. Fall back to WebFetch with a strict, structured prompt — and flag the resulting URLs as "verify manually" since WebFetch can occasionally infer URLs.
- **Parallel API requests above ~5 trigger 403 on Ashby**. Process Ashby slugs sequentially with `sleep 1`; the other ATSes are fine in parallel.
- **Location strings need keyword matching, not regex**. Strings like `"Remote, US-Southeast"` or `"Amsterdam, Netherlands"` are messy. A small keyword list (EMEA cities, US-only signals, strong markers like "Europe"/"EMEA") with explicit precedence rules beats a single regex every time.
- **A "Senior Technical PM" is not a "Senior PM"**. If your filter blocks `technical` you also block titles like "Senior Technical Product Manager" which are usually within scope. Distinguish `technical program` (TPM) from `technical product` (a real PM).

See [`docs/lessons-learned.md`](docs/lessons-learned.md) for the long version.

## How to use this

This was built for [Claude Code](https://claude.com/claude-code). The three `.md` files in `commands/` are slash-command definitions — drop them into your `~/.claude/commands/` directory and they become `/scan-targets`, `/evaluate-target`, `/radar`.

To adapt to your case, edit:

- `commands/scan-targets.md` — the inventory file path, the title filter (`TITLE_BLOCKERS`), the location filter (`SPAIN_STRONG`, `EMEA_PLACES`, `US_ONLY_PLACES`)
- `commands/evaluate-target.md` — the user profile location, the scoring weights, what counts as "Aplicar YA" / "Decidir" / "Descartar"
- `commands/radar.md` — the cron in `/schedule` once you're happy with it

An empty inventory template is in [`examples/targets-inventory.example.md`](examples/targets-inventory.example.md).

## Scheduling

Once you trust the output, schedule it weekly via Claude Code's `/schedule`:

```
/schedule
# name: radar-semanal
# cron: 0 8 * * 1     (Monday 8am, your timezone)
# command: /radar
```

## License

MIT. See [LICENSE](LICENSE).

## Acknowledgments

Built with Claude Code (Anthropic). The pattern was iterated through real use against a real job search — the code only got robust because real URLs went stale, real APIs rate-limited, and a real ranking surfaced real candidates.
