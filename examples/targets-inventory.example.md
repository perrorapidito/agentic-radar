# Targets Inventory — Example

This is the file `scan-targets` reads. Two sections matter to the scanner:

## ATS Slug Map (auto-populated by scans)

The scanner reads this first and skips slug discovery for any company listed here. New discoveries get appended at the end of each scan.

```
ExampleCo       greenhouse:exampleco
SomeStartup     ashby:somestartup
LargeEnterprise workday:largeent.wd5/External_Careers
SmallEU         lever:smalleu
TinyCo          personio:tinyco
DormantCorp     none                            # known no public ATS — fallback WebFetch
```

## Inventory tables (your targets, organized however you want)

The scanner parses each row for: company name + careers URL. Anything else is for your own organization.

### Tier 1 — Strong fit

| # | Company | Sector | HQ | Status | Headcount | Remote+location evidence | Careers URL |
|---|---|---|---|---|---|---|---|
| 1 | ExampleCo | SaaS/AI | San Francisco | Series C | ~300 | Remote-first, EMEA via EOR | https://exampleco.com/careers |
| 2 | SomeStartup | DevTools | Berlin | Series A | ~80 | Hybrid Berlin, remote DACH | https://jobs.ashbyhq.com/somestartup |

### Tier 2 — Moderate fit

| # | Company | Sector | HQ | Status | Headcount | Remote+location evidence | Careers URL |
|---|---|---|---|---|---|---|---|
| 3 | LargeEnterprise | Enterprise SaaS | Boston | Public | ~10,000 | Global hybrid, Madrid hub | https://largeent.wd5.myworkdayjobs.com/External_Careers |

### Tier 3 — Stretch / opportunistic

| # | Company | Sector | HQ | Status | Headcount | Remote+location evidence | Careers URL |
|---|---|---|---|---|---|---|---|
| 4 | SmallEU | FinTech | Madrid | Bootstrapped | ~50 | HQ Madrid, fully remote | https://jobs.lever.co/smalleu |

---

## Notes on format

- The "Tier" semantics are yours to define. The scanner doesn't enforce a meaning; it just lets you filter via `/scan-targets only:tier1`.
- The ATS Slug Map is the only structural requirement. Without it, every scan re-discovers slugs via WebSearch, which is wasteful.
- Discovered slugs get persisted automatically at the end of each scan.

## How to seed this for your case

1. Copy this file to `./inventory/targets.md`.
2. Replace the example rows with your actual target companies.
3. Run `/scan-targets` once. The first run will WebSearch each company to discover its ATS slug; subsequent runs will use the map.
4. Manually correct any wrong slugs the first run inferred — they're persisted as `none` if not found, or as `ats:slug` if found.
