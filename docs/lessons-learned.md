# Lessons learned

Real bugs and design pivots from building this. The kind of detail you only get from running it against live APIs.

## Ashby renders with JavaScript

The HTML payload at `jobs.ashbyhq.com/{slug}` is a skeleton. Real job listings come from `https://api.ashbyhq.com/posting-api/job-board/{slug}?includeCompensation=true`. WebFetch on the careers page returns nothing useful — go to the JSON API.

## Ashby `locationName` is often empty

The structured location lives in `workplaceType` (string: "Remote" / "Hybrid" / "On-site") + `secondaryLocations[].location` (list of cities/regions) + `isRemote` (boolean). A filter checking only `locationName` drops legitimate remote hits. Build the location string from all four fields and run the verdict against the concatenated form.

## Workable's v3 endpoint returns 404

The path `/api/v3/accounts/{slug}/jobs` is documented in old tutorials but no longer exists. The working endpoint is:

```
GET https://apply.workable.com/api/v1/widget/accounts/{slug}
```

Returns `{name, description, jobs: [{title, location, country, shortcode, ...}]}`. The job URL is constructed as `https://apply.workable.com/{slug}/j/{shortcode}`.

## Greenhouse redirects deleted jobs to `{slug}?error=true`

A request to a deleted job ID returns 200 OK with content-type `text/html`, but the effective URL has changed to `https://job-boards.greenhouse.io/{slug}?error=true`. HEAD-validating only the status code accepts dead jobs as live. Validate the *effective URL*:

```bash
curl -sL -o /dev/null -w "%{http_code}|%{url_effective}" --max-time 12 "$url"
```

Check that `%{url_effective}` does not contain `?error=true` and does not equal the board root.

## Workday CXS API is anti-bot aggressive

The endpoint `POST /wday/cxs/{tenant}/{site}/jobs` accepts JSON bodies and returns structured job data. First request often works. Second and subsequent return HTTP 400 or 422 with `errorCode: HTTP_400` / `HTTP_422` — the tenant's IDS detected scripted access. Cookie-priming via a prior GET sometimes helps; sometimes it makes it worse.

Observed across multiple Workday tenants on the `*.myworkdayjobs.com` and `wd3.myworkdaysite.com` domains:

- First POST to `/wday/cxs/{tenant}/{site}/jobs` with a JSON body returns valid `jobPostings`.
- Second POST returns `errorCode: HTTP_400` or `HTTP_422`.
- Site-name guessing is itself unreliable — common values (`External`, `External_Careers`, `External_Career_Site`, the company name) can each be valid for some tenants and 404 for others.

For production use, fall back to WebFetch with a strict prompt instead of relying on CXS.

## Parallel API requests above ~5 trigger 403 on Ashby

Running 25 Ashby slugs in parallel via Python `urllib` produces 403 responses. The first ~5 succeed, the rest get blocked. With a `User-Agent: Mozilla/5.0` header and `sleep 1` between requests, all 25 succeed. The other ATSes (Greenhouse, Lever, Workable) tolerate parallel batches of 5+ without issue.

## Location strings need keyword matching, not regex

Real location strings:
- `"Remote, US-Southeast"` — US-only despite the word "Remote"
- `"Amsterdam, Netherlands"` — on-site Amsterdam
- `"Netherlands (remote)"` — remote anywhere in NL
- `"United Kingdom (remote)"` — remote anywhere in UK
- `"Hybrid | Remote | London | New York | Montreal"` — split fields concatenated
- `""` — empty (Ashby)
- `"Remote | Remote | Europe"` — Europe-eligible

A single regex can't handle this taxonomy. Better: explicit keyword lists with precedence:

1. Strong "eligible everywhere" markers (`spain`, `europe`, `emea`, `worldwide`, `anywhere`) → OK
2. Single EMEA country/city + `remote` keyword → contractor-viable (EMEA_CTRY)
3. Single EMEA city without `remote` → on-site only (NO)
4. US/Canada/India/APAC city markers → NO
5. Bare `remote` with no region → AMBIG (user verifies)

## URL hallucination — the silent failure mode

Before HEAD validation: when the AI builds reports from scraped HTML, it occasionally infers a plausible URL pattern (e.g. `boards.greenhouse.io/{slug}/jobs/{some-id}`) rather than extracting the actual `absolute_url` field. ~10% of these inferred URLs don't exist, and the report ships dead links.

Two defenses combined eliminate this:
1. Use the JSON API and copy the `absolute_url` / `hostedUrl` / `jobUrl` / `applyUrl` field **verbatim**. Never construct.
2. HEAD-validate every URL before reporting it.

## WebSearch lag is 1–4 weeks

A common naive approach: WebSearch `"Company X product manager job"` → click the first result. Looks correct in the moment, but Google indexed that page when the job was active. By the time you fetch it, the job may be closed for a week or longer.

WebSearch is only useful for **slug discovery** (which ATS does the company use — stable signal), never for finding specific job URLs.

## "Senior Technical PM" is not "Senior Program Manager"

When building the title block-list, `technical program` (TPM) is a distinct role from `technical product` (a real PM role at the senior level). A naive filter blocking `technical` drops legitimate "Senior Technical Product Manager" titles. Use phrase-level matching, not word-level.

## Subagent context isolation is the actual win

Running 7 deep-evaluations inline in the main context: ~50K tokens of WebSearch+JD+scoring polluting subsequent reasoning.

Running them as 7 subagents in batches of 3 in parallel: ~3K tokens in main context (just the verdict blocks), 50K total token spend but spread across isolated contexts that get garbage-collected.

The token cost is similar; the *quality of subsequent reasoning* in the main thread is dramatically better because the context isn't crowded with raw scraping output. That's the real value of subagent orchestration for long-tail evaluation.

## Human-in-the-loop for irreversible state

The orchestrator can fully automate **read** operations (scan, evaluate, rank). It cannot — and should not — fully automate **write to pipeline** because pipeline state is what the human acts on. An auto-added entry can't be un-added in the user's head; a false negative is invisible.

A useful invariant: subagents return verdict blocks; the orchestrator presents a ranked list; the human says `1,3,5`; only then does the pipeline get written. This is the inverse of how most "AI agent" pitches frame it (full autonomy), and it's what makes the system trustworthy enough to run weekly.
