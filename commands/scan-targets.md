# Scan Targets — Discover open opportunities across an inventory

Reads a target inventory file (`./inventory/targets.md` by default) and finds open positions matching strict title and location filters. Uses ATS JSON APIs as the primary discovery mechanism; falls back to WebFetch for proprietary career sites.

Built for the job-search domain (Product Manager / Senior PM, Spain-viable locations). Adapt the title and location filter lists to your domain.

## Design principles

1. **JSON APIs before scraping**. Greenhouse, Lever, Ashby, Workable and Personio expose canonical JSON boards. Eliminate URL hallucination, ~10× cheaper in tokens than HTML.
2. **Validate before reporting**. No URL is reported without a final HEAD request confirming 200 + non-error effective URL.
3. **WebSearch only discovers slugs, never URLs**. Search engines are 1–4 weeks behind real ATS state.
4. **Persist ATS knowledge**. When you discover `ats:slug` for a target, write it back to the inventory so future scans skip discovery.

## Title filter (strict)

Include titles containing `product manager` AND NOT containing any of:

```
principal       staff           lead          group        director
vp              head of         marketing     technical program
assistant       associate       intern        contract     manager of product
```

This admits:
- ✅ `Product Manager`, `Senior Product Manager`, `Sr. Product Manager`, `Senior Technical Product Manager`
- ❌ `Principal/Staff/Lead/Group PM`, `Director/Head/VP of Product`, `Product Marketing Manager`, `Technical Program Manager`

## Execution

### Step 1 — Read and parse inventory

Read `./inventory/targets.md`:

1. **"ATS Slug Map" section** (top): load `Company  ats:slug` lines as a dictionary. These targets skip directly to Step 2.
2. **Inventory tables**: for each target NOT in the map, read Name + Careers URL → mark for Step 1b (discovery).

If `$args` contains `only:<X>`, filter the inventory accordingly. Without args, scan everything.

#### Step 1b — Discover ATS (only if not in map)

Single WebSearch: `"{company}" careers (greenhouse OR lever OR ashby OR workable OR workday OR personio)`.

Extract the slug:
- `boards.greenhouse.io/{slug}` or `job-boards.greenhouse.io/{slug}` → `greenhouse:{slug}`
- `jobs.lever.co/{slug}` → `lever:{slug}`
- `jobs.ashbyhq.com/{slug}` → `ashby:{slug}`
- `apply.workable.com/{slug}` → `workable:{slug}`
- `{tenant}.myworkdayjobs.com/{site}` → `workday:{tenant}.{wd1|wd3|wd5}` + remember `site`
- `{slug}.jobs.personio.com` → `personio:{slug}`
- nothing → `none` (WebFetch fallback)

At the end of Step 5, persist discovered slugs back to the inventory.

### Step 2 — Fetch by ATS

Process in parallel batches of 5 for Greenhouse/Lever/Workable. **Ashby must be sequential** (403 rate-limit above ~5 concurrent).

Shared filter snippet (used inside each handler):

```python
TITLE_BLOCKERS = ['principal','staff','lead','group','director','vp ','head of',
                  'marketing','technical program','assistant','associate',
                  'intern','contract','manager of product']
def is_target(t):
    tl = t.lower()
    return 'product manager' in tl and not any(x in tl for x in TITLE_BLOCKERS)
```

#### Greenhouse → `greenhouse:{slug}`

```
GET https://boards-api.greenhouse.io/v1/boards/{slug}/jobs
```

Returns `{jobs: [{id, title, location: {name}, absolute_url}, ...]}`. Filter on `title`, extract `absolute_url` verbatim.

#### Lever → `lever:{slug}`

```
GET https://api.lever.co/v0/postings/{slug}?mode=json
```

Returns an array of `{id, text (title), categories: {location}, hostedUrl}`. Filter on `text`, also accept `product owner` (Lever convention).

#### Ashby → `ashby:{slug}` (SEQUENTIAL with sleep 1s)

```
GET https://api.ashbyhq.com/posting-api/job-board/{slug}?includeCompensation=true
```

Build the location from `locationName + workplaceType + isRemote + secondaryLocations[].location`. `locationName` is often empty on remote jobs — don't filter on it alone.

#### Workable → `workable:{slug}`

```
GET https://apply.workable.com/api/v1/widget/accounts/{slug}
```

Note: `/api/v3/accounts/{slug}/jobs` returns 404. Use the v1 widget endpoint.

#### Personio → `personio:{slug}`

```
GET https://{slug}.jobs.personio.com/xml
```

Returns XML feed with `<position>` elements. Parse `name`, `office`, `recruitingCategory`.

#### Workday → `workday:{tenant}.{wdN}`

The CXS API (`/wday/cxs/{tenant}/{site}/jobs`) exists but is anti-bot aggressive: first request often works, subsequent ones return 400/422. Not reliable.

**Strategy**: WebFetch on the careers URL with a strict prompt:

> "List ALL currently open Product Manager or Senior Product Manager roles. For each return one line in pipe format: `title|location|direct_url`. Exclude Principal, Staff, Lead, Group, Director, VP, Head of, Product Marketing, TPM. Only include roles where location explicitly mentions: Spain, Madrid, Barcelona, Valencia, Remote EMEA, Remote Europe, Worldwide, or Anywhere. If none qualify, output exactly: `NONE`."

Flag resulting URLs as ⚠️ "via WebFetch — verify at apply time".

#### No known ATS (`none`) → WebFetch fallback

Same prompt as Workday, against the inventory's careers URL.

### Step 3 — Location verdict

For every position the handlers return, classify:

- **OK** ✅: contains `spain`, `españa`, `madrid`, `barcelona`, `valencia`, ` emea`, `europe`, `europa`, `worldwide`, `anywhere`, `global remote`, `utc+`, `emea/amer`, `any location`
- **EMEA_CTRY** ⚠️ (contractor-viable): single EMEA country/city (Netherlands, Amsterdam, UK, London, Germany, Berlin, France, Paris, Ireland, Portugal, Poland, Sweden, ...) **+** the word `remote`
- **AMBIG** ⚠️ (user must verify): just `remote` with no region, or empty `locationName`
- **NO** 🚫 → drop: US/Canada/LATAM/APAC/India only, or EMEA city without `remote` (on-site mandatory)

### Step 4 — HEAD validation (mandatory gate)

Before reporting any URL, HEAD it in parallel:

```bash
for url in $URLS; do
  (code=$(curl -sL -o /dev/null -w "%{http_code}|%{url_effective}" \
    -A "Mozilla/5.0" --max-time 12 "$url"); echo "$url|$code") &
done
wait
```

Rules:
- `%{http_code}` ≠ 200 → drop, move to "verified-closed"
- `%{url_effective}` contains `?error=true` (Greenhouse pattern) or redirects to board root without job ID (Ashby pattern) → drop
- 403 (anti-bot) → keep with "verify manually" flag

### Step 5 — Report and persist

#### 5a — Chat report

```
## Positions found — [date]

| Source | Company | Role | Location | Verdict | URL |
|---|---|---|---|---|---|
| T1 | X | Senior PM — Y | Remote Europe | ✅ OK | url |

## No active matches
- Company A — board active, 0 PMs that pass seniority filter
- Company B — no results

## Verified closed (dead URLs)
- Company C — 404 / redirect to error
```

#### 5b — Update `./pipeline/active-opportunities.md`

1. **Add** new positions in the existing format.
2. **Move to "Archived"** positions whose URLs failed HEAD.
3. **Dedup** by exact URL before adding.
4. **Update header**: last-updated date + scan-history row.

#### 5c — Persist ATS slugs discovered in Step 1b

Write any newly-discovered `Company  ats:slug` line to the inventory's "ATS Slug Map" section. This is what makes future scans cheaper.

### Step 6 — Close

> "Want me to deep-evaluate any of these with `/evaluate-target`, or add them straight to the pipeline?"

## Optional args

- **No args** → scan the entire inventory. Default.
- `/scan-targets only:<tier>` — restrict to one tier
- `/scan-targets only:<company>` — restrict to one company

## Critical notes

- **Never** construct URLs by pattern. Only copy verbatim from JSON or HEAD-validated.
- **Never** use WebSearch for specific job URLs — only for discovering ATS slugs of new companies.
- The title filter is **strict** by design. To widen it, edit `TITLE_BLOCKERS`.
- If a careers URL stays 404 between scans, archive and flag with ⚠️ in the inventory (don't delete — it can come back).
