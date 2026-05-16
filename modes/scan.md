# Mode: scan -- Portal Scanner (Job Discovery)

Scan configured job portals, filter by title relevance, and add new jobs to the
pipeline for later evaluation.

> **Note (v1.5+):** The default scanner (`scan.mjs` / `npm run scan`) is
> **zero-token** and only queries the public Greenhouse, Ashby, and Lever APIs
> directly. The Playwright/WebSearch levels described below are the **agent**
> flow (run by Claude/Codex), not what `scan.mjs` does. If a company does not
> have a Greenhouse/Ashby/Lever API, `scan.mjs` will ignore it; for those cases,
> the agent should complete Level 1 (Playwright) or Level 3 (WebSearch)
> manually.

## Recommended Execution

Run as a subagent to avoid consuming the main context:

```text
Agent(
    subagent_type="general-purpose",
    prompt="[content of this file + specific data]",
    run_in_background=True
)
```

## Configuration

Read `portals.yml`, which contains:
- `search_queries`: WebSearch queries with `site:` filters by portal for broad discovery
- `tracked_companies`: specific companies with `careers_url` for direct navigation
- `title_filter`: positive/negative/seniority_boost keywords for title filtering

## Discovery Strategy (3 Levels)

### Level 1 -- Direct Playwright (PRIMARY)

**For each company in `tracked_companies`:** Navigate to its `careers_url` with
Playwright (`browser_navigate` + `browser_snapshot`), read ALL visible job
listings, and extract title + URL for each one. This is the most reliable
method because:
- It sees the page in real time, not cached Google results.
- It works with SPAs such as Ashby, Lever, and Workday.
- It detects new jobs immediately.
- It does not depend on Google indexing.

**Every company MUST have `careers_url` in portals.yml.** If it does not, find
it once, save it, and use it in future scans.

### Level 2 -- ATS APIs / Feeds (COMPLEMENTARY)

For companies with a public API or structured feed, use the JSON/XML response
as a fast complement to Level 1. This is faster than Playwright and reduces
visual scraping errors.

**Current support (variables in `{}`):**
- **Greenhouse**: `https://boards-api.greenhouse.io/v1/boards/{company}/jobs`
- **Ashby**: `https://jobs.ashbyhq.com/api/non-user-graphql?op=ApiJobBoardWithTeams`
- **BambooHR**: list `https://{company}.bamboohr.com/careers/list`; job detail `https://{company}.bamboohr.com/careers/{id}/detail`
- **Lever**: `https://api.lever.co/v0/postings/{company}?mode=json`
- **Teamtailor**: `https://{company}.teamtailor.com/jobs.rss`
- **Workday**: `https://{company}.{shard}.myworkdayjobs.com/wday/cxs/{company}/{site}/jobs`

**Provider parsing convention:**
- `greenhouse`: `jobs[]` -> `title`, `absolute_url`
- `ashby`: GraphQL `ApiJobBoardWithTeams` with `organizationHostedJobsPageName={company}` -> `jobBoard.jobPostings[]` (`title`, `id`; build public URL if missing from payload)
- `bamboohr`: list `result[]` -> `jobOpeningName`, `id`; build detail URL `https://{company}.bamboohr.com/careers/{id}/detail`; to read the full JD, GET the detail endpoint and use `result.jobOpening` (`jobOpeningName`, `description`, `datePosted`, `minimumExperience`, `compensation`, `jobOpeningShareUrl`)
- `lever`: root array `[]` -> `text`, `hostedUrl` (fallback: `applyUrl`)
- `teamtailor`: RSS items -> `title`, `link`
- `workday`: `jobPostings[]`/`jobPostings` depending on tenant -> `title`, `externalPath` or URL built from the host

### Level 3 -- WebSearch Queries (BROAD DISCOVERY)

`search_queries` with `site:` filters cover portals horizontally (all Ashby, all
Greenhouse, etc.). This is useful for discovering NEW companies that are not
yet in `tracked_companies`, but results can be stale.

**Execution priority:**
1. Level 1: Playwright -> all `tracked_companies` with `careers_url`
2. Level 2: API -> all `tracked_companies` with `api:`
3. Level 3: WebSearch -> all `search_queries` with `enabled: true`

The levels are additive: run all of them, merge results, and deduplicate.

## Workflow

1. **Read configuration**: `portals.yml`
2. **Read history**: `data/scan-history.tsv` -> URLs already seen
3. **Read dedup sources**: `data/applications.md` + `data/pipeline.md`

4. **Level 1 -- Playwright scan** (parallel in batches of 3-5):
   For each company in `tracked_companies` with `enabled: true` and a defined `careers_url`:
   a. `browser_navigate` to the `careers_url`
   b. `browser_snapshot` to read all job listings
   c. If the page has filters/departments, navigate relevant sections
   d. For each job listing extract: `{title, url, company}`
   e. If the page paginates results, navigate additional pages
   f. Accumulate into the candidate list
   g. If `careers_url` fails (404, redirect), try `scan_query` as fallback and note that the URL needs updating

5. **Level 2 -- ATS APIs / feeds** (parallel):
   For each company in `tracked_companies` with `api:` defined and `enabled: true`:
   a. WebFetch the API/feed URL
   b. If `api_provider` is defined, use its parser; if not, infer by domain (`boards-api.greenhouse.io`, `jobs.ashbyhq.com`, `api.lever.co`, `*.bamboohr.com`, `*.teamtailor.com`, `*.myworkdayjobs.com`)
   c. For **Ashby**, send POST with:
      - `operationName: ApiJobBoardWithTeams`
      - `variables.organizationHostedJobsPageName: {company}`
      - GraphQL query for `jobBoardWithTeams` + `jobPostings { id title locationName employmentType compensationTierSummary }`
   d. For **BambooHR**, the list only returns basic metadata. For each relevant item, read `id`, GET `https://{company}.bamboohr.com/careers/{id}/detail`, and extract the full JD from `result.jobOpening`. Use `jobOpeningShareUrl` as the public URL if present; otherwise use the detail URL.
   e. For **Workday**, send POST JSON with at least `{"appliedFacets":{},"limit":20,"offset":0,"searchText":""}` and paginate by `offset` until results are exhausted
   f. For each job, extract and normalize: `{title, url, company}`
   g. Accumulate into the candidate list, deduping against Level 1

6. **Level 3 -- WebSearch queries** (parallel when possible):
   For each query in `search_queries` with `enabled: true`:
   a. Run WebSearch with the defined `query`
   b. From each result extract: `{title, url, company}`
      - **title**: from the result title before " @ " or " | "
      - **url**: result URL
      - **company**: after " @ " in the title, or extracted from the domain/path
   c. Accumulate into the candidate list, deduping against Levels 1+2

6. **Filter by title** using `title_filter` from `portals.yml`:
   - At least 1 keyword from `positive` must appear in the title (case-insensitive)
   - 0 keywords from `negative` must appear
   - `seniority_boost` keywords add priority but are not required

6b. **Filter by location (optional)** using `location_filter` from `portals.yml`:
   - If the `location_filter` block is absent, all locations pass by default
   - Empty location in a job -> pass; do not penalize missing data
   - Any keyword from `block` present -> reject; block takes precedence over allow
   - Empty `allow` -> pass if block did not match
   - Non-empty `allow` -> must match at least one keyword
   - All matches are case-insensitive substrings
   - Persist location as the 7th column in `scan-history.tsv` for later auditing

7. **Deduplicate** against 3 sources:
   - `scan-history.tsv` -> exact URL already seen
   - `applications.md` -> normalized company + role already evaluated
   - `pipeline.md` -> exact URL already pending or processed

7.5. **Verify liveness for WebSearch results (Level 3)** -- BEFORE adding to pipeline:

   WebSearch results can be stale because Google caches results for weeks or
   months. To avoid evaluating expired jobs, verify every new URL from Level 3
   with Playwright. Levels 1 and 2 are inherently real-time and do not need this
   verification.

   For each new Level 3 URL (sequential -- NEVER run Playwright in parallel):
   a. `browser_navigate` to the URL
   b. `browser_snapshot` to read the content
   c. Classify:
      - **Active**: visible job title + role description + visible Apply/Submit control in the main content. Do not count generic header/navbar/footer text.
      - **Expired** if any of these signals appear:
        - Final URL contains `?error=true` (Greenhouse redirects this way when a job is closed)
        - Page contains: "job no longer available" / "no longer open" / "position has been filled" / "this job has expired" / "page not found"
        - Only navbar and footer are visible, with no JD content (content < ~300 chars)
   d. If expired: record in `scan-history.tsv` with status `skipped_expired` and discard
   e. If active: continue to Step 8

   **Do not interrupt the whole scan if one URL fails.** If `browser_navigate`
   errors (timeout, 403, etc.), mark it as `skipped_expired` and continue with
   the next URL.

8. **For every verified new job that passes filters**:
   a. Add to the `Pending` section of `pipeline.md`: `- [ ] {url} | {company} | {title}`
   b. Record in `scan-history.tsv`: `{url}\t{date}\t{query_name}\t{title}\t{company}\tadded`

9. **Jobs filtered by title**: record in `scan-history.tsv` with status `skipped_title`
10. **Duplicate jobs**: record with status `skipped_dup`
11. **Expired jobs (Level 3)**: record with status `skipped_expired`

## Extracting Title and Company from WebSearch Results

WebSearch results usually come in formats like: `"Job Title @ Company"`,
`"Job Title | Company"`, or `"Job Title -- Company"`.

Extraction patterns by portal:
- **Ashby**: `"Senior AI PM (Remote) @ EverAI"` -> title: `Senior AI PM`, company: `EverAI`
- **Greenhouse**: `"AI Engineer at Anthropic"` -> title: `AI Engineer`, company: `Anthropic`
- **Lever**: `"Product Manager - AI @ Temporal"` -> title: `Product Manager - AI`, company: `Temporal`

Generic regex: `(.+?)(?:\s*[@|--–-]\s*|\s+at\s+)(.+?)$`

## Private URLs

If a non-public URL is found:
1. Save the JD in `jds/{company}-{role-slug}.md`
2. Add it to pipeline.md as: `- [ ] local:jds/{company}-{role-slug}.md | {company} | {title}`

## Scan History

`data/scan-history.tsv` tracks ALL URLs seen:

```text
url	first_seen	portal	title	company	status
https://...	2026-02-10	Ashby -- AI PM	PM AI	Acme	added
https://...	2026-02-10	Greenhouse -- SA	Junior Dev	BigCo	skipped_title
https://...	2026-02-10	Ashby -- AI PM	SA AI	OldCo	skipped_dup
https://...	2026-02-10	WebSearch -- AI PM	PM AI	ClosedCo	skipped_expired
```

## Output Summary

```text
Portal Scan -- {YYYY-MM-DD}
===========================
Queries run: N
Jobs found: N total
Filtered by title: N relevant
Duplicates: N (already evaluated or in pipeline)
Expired discarded: N (dead links, Level 3)
New jobs added to pipeline.md: N

  + {company} | {title} | {query_name}
  ...

-> Run /career-ops pipeline to evaluate the new jobs.
```

## careers_url Management

Every company in `tracked_companies` must have `careers_url`: the direct URL to
its jobs page. This avoids searching for it every time.

**RULE: Always use the company's corporate URL; use the ATS endpoint only if no
corporate careers page exists.**

The `careers_url` should point to the company's own careers page whenever
available. Many companies use Workday, Greenhouse, or Lever underneath, but
expose vacancy IDs only through their corporate domain. Using the direct ATS URL
when a corporate page exists can cause false 410 errors because job IDs do not
match.

| Correct (corporate) | Incorrect as first choice (direct ATS) |
|---|---|
| `https://careers.mastercard.com` | `https://mastercard.wd1.myworkdayjobs.com` |
| `https://openai.com/careers` | `https://job-boards.greenhouse.io/openai` |
| `https://stripe.com/jobs` | `https://jobs.lever.co/stripe` |

Fallback: if you only have the direct ATS URL, first navigate to the company's
website and locate its corporate careers page. Use the direct ATS URL only if
the company has no corporate careers page.

**Known platform patterns:**
- **Ashby:** `https://jobs.ashbyhq.com/{slug}`
- **Greenhouse:** `https://job-boards.greenhouse.io/{slug}` or `https://job-boards.eu.greenhouse.io/{slug}`
- **Lever:** `https://jobs.lever.co/{slug}`
- **BambooHR:** list `https://{company}.bamboohr.com/careers/list`; detail `https://{company}.bamboohr.com/careers/{id}/detail`
- **Teamtailor:** `https://{company}.teamtailor.com/jobs`
- **Workday:** `https://{company}.{shard}.myworkdayjobs.com/{site}`
- **Custom:** the company's own URL, e.g. `https://openai.com/careers`

**API/feed patterns by platform:**
- **Ashby API:** `https://jobs.ashbyhq.com/api/non-user-graphql?op=ApiJobBoardWithTeams`
- **BambooHR API:** list `https://{company}.bamboohr.com/careers/list`; detail `https://{company}.bamboohr.com/careers/{id}/detail` (`result.jobOpening`)
- **Lever API:** `https://api.lever.co/v0/postings/{company}?mode=json`
- **Teamtailor RSS:** `https://{company}.teamtailor.com/jobs.rss`
- **Workday API:** `https://{company}.{shard}.myworkdayjobs.com/wday/cxs/{company}/{site}/jobs`

**If `careers_url` does not exist** for a company:
1. Try the known pattern for its platform
2. If that fails, run a quick WebSearch: `"{company}" careers jobs`
3. Navigate with Playwright to confirm it works
4. **Save the found URL in portals.yml** for future scans

**If `careers_url` returns 404 or redirects:**
1. Note it in the output summary
2. Try `scan_query` as fallback
3. Mark it for manual update

## portals.yml Maintenance

- **ALWAYS save `careers_url`** when a new company is added
- Add new queries as you discover interesting portals or roles
- Disable queries with `enabled: false` if they generate too much noise
- Adjust filtering keywords as target roles evolve
- Add companies to `tracked_companies` when they are worth monitoring closely
- Verify `careers_url` periodically because companies change ATS platforms
