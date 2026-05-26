# z21-jobs

Auto-updating jobs board that scrapes open roles from Z21 Fund II portfolio companies and publishes them to [z21-jobs.vercel.app](https://z21-jobs.vercel.app/). Refreshes every Monday at 6 AM UTC with no manual effort.

---

## Purpose & Workflow

The board exists so the Z21 team and the wider network can browse open roles across portfolio companies in one place, without chasing individual careers pages.

A Python scraper (`scripts/scrape.py`) runs weekly via GitHub Actions. It pulls live job listings from each company's ATS, normalizes the data into a common schema, and writes everything to `data/jobs.json`. GitHub Actions then commits that file back to the repo. Vercel detects the new commit and redeploys the Next.js frontend automatically.

The board is live at: **https://z21-jobs.vercel.app/**

---

## Step 1: Initial System Setup

Only needed if you want to run the scraper or frontend locally.

**Prerequisites:**
- Node.js 20+
- Python 3.11+
- `pip`

```bash
npm install
pip install requests
```

---

## Step 2: Adding a Company

The company roster lives in `data/companies.json`. Each entry tells the scraper which company to fetch and which adapter to use.

To add a new company, append an object to that file:

```json
{
  "name": "Acme Corp",
  "url": "https://boards.greenhouse.io/acmecorp",
  "type": "greenhouse",
  "website": "https://acme.com/",
  "description": "One-line company description"
}
```

**Supported `type` values:**

| `type` | Platform | Notes |
|---|---|---|
| `workable` | Workable | Parses `apply.workable.com/<slug>/jobs.md` |
| `yc` | Y Combinator | Parses embedded `data-page` JSON on the YC company page |
| `teamtailor` | TeamTailor | Reads the `/jobs.rss` feed |
| `greenhouse` | Greenhouse | Calls `boards-api.greenhouse.io/v1/boards/<slug>/jobs`; add optional `"greenhouse_slug"` if slug differs from the URL |
| `lever` | Lever | Calls `api.lever.co/v0/postings/<slug>?mode=json` |
| `notion_collection` | Notion database board | Requires `notion_collection_id` and `notion_view_id` fields |
| `notion_toggles` | Notion toggle-block page | Requires `notion_page_id` field |
| `zime_html` | Custom HTML (Zime format) | Regex-based; fragile on redesign |
| `devicethread_html` | Custom HTML (Devicethread format) | Regex-based; fragile on redesign |
| `link` | Link-only | Appears in "More Portfolio Companies" section; no scraping |

For platforms not in this list, a developer needs to write a new adapter function in `scripts/scrape.py` and add it to the `SCRAPERS` dict.

---

## Step 3: Running the Scraper

**Automatic:** The scraper runs every Monday at 6 AM UTC. It commits any changes to `data/jobs.json` and pushes them to `main`. Vercel redeploys within a few minutes.

**Manual trigger from GitHub:** Go to the [Actions tab](https://github.com/UditBatish/z21-jobs/actions), select "Scrape Jobs", click "Run workflow".

**Local run:**

```bash
python scripts/scrape.py
```

Example terminal output:

```
[WORKABLE] Scraping Origin...
  ✓ 14 jobs
[YC] Scraping Lighthouz AI...
  ✓ 3 jobs
[TEAMTAILOR] Scraping Greenjets...
  ✓ 11 jobs
[GREENHOUSE] Scraping RISA Labs...
  ✓ 6 jobs
[SKIP] Coverself — unknown platform 'link'

Done. 47 total jobs → data/jobs.json
```

The `[SKIP]` line for `link`-type companies is expected. Only scraped jobs go to `data/jobs.json`.

---

## Step 4: Deploying Updates

No manual deployment step exists. Once the scraper commits `data/jobs.json` to `main`, Vercel picks it up and rebuilds the site automatically.

To preview the frontend locally before pushing any frontend changes:

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000). The page reads `data/jobs.json` at build time, so local job data reflects whatever is currently in that file.

---

## Troubleshooting and Safety Tips

**1. A company shows 0 jobs after the scrape runs.**
Check the `type` field in `data/companies.json`. A typo causes the scraper to print `[SKIP] <Company> — unknown platform '<type>'` and move on. Fix the typo and re-run.

**2. Greenhouse scraper fails with 404.**
The slug in the careers URL sometimes differs from the actual Greenhouse board slug. Find the correct slug by opening DevTools on the company's Greenhouse board, filtering network requests for `boards-api.greenhouse.io`, and reading the slug from the request URL. Add it explicitly as `"greenhouse_slug": "<slug>"` in `companies.json`.

**3. Notion scraper returns 0 jobs.**
The `notion_collection_id`, `notion_view_id`, or `notion_page_id` values may have changed if the Notion page was recreated or duplicated. Re-derive the IDs by opening DevTools on the Notion page, triggering a `queryCollection` or `loadPageChunk` network request, and copying the IDs from the request payload.

**4. GitHub Actions fails with a permission error on `git push`.**
The repo needs "Read and write permissions" enabled for Actions. Go to the repo on GitHub: Settings > Actions > General > Workflow permissions > select "Read and write permissions" > Save.

**5. Custom HTML scraper breaks after a company redesigns their careers page.**
The `zime_html` and `devicethread_html` scrapers use regex against raw HTML. If the markup changes, the regex stops matching and returns 0 jobs silently. Update the regex patterns in `scrape_zime_html()` or `scrape_devicethread_html()` inside `scripts/scrape.py` to match the new structure.

---

## Technical Architecture

```
data/companies.json  (config: company roster + platform type)
        │
        ▼
scripts/scrape.py  (dispatcher: iterates companies, calls platform adapter)
        │
        ├── scrape_workable()           → Workable jobs.md endpoint
        ├── scrape_yc()                 → YC page embedded JSON
        ├── scrape_teamtailor()         → TeamTailor RSS feed
        ├── scrape_greenhouse()         → Greenhouse boards API v1
        ├── scrape_lever()              → Lever postings API v0
        ├── scrape_notion_collection()  → Notion internal queryCollection API
        ├── scrape_notion_toggles()     → Notion internal loadPageChunk API
        ├── scrape_zime_html()          → HTML regex extraction
        └── scrape_devicethread_html()  → HTML regex extraction
                │
                ▼
        data/jobs.json  (output: flat list of all scraped jobs)
                │
                ▼
        app/page.tsx  (Next.js Server Component: reads jobs.json at build time)
                │
                ▼
        components/JobBoard.tsx  (Client Component: search + filter UI)
                │
                ├── search (title, department, location keywords)
                ├── filter by company
                ├── filter by region  (India / United States / Remote / Rest of World)
                └── filter by function  (Engineering / Product / Sales & Success / …)
                │
                ▼
        https://z21-jobs.vercel.app/
```

### Scraper dispatch (`scripts/scrape.py`)

The `main()` function iterates over `data/companies.json` and dispatches each entry to the correct adapter via the `SCRAPERS` dict (a mapping of `type` string to function). The `"link"` type is intentionally absent from `SCRAPERS`: link-only companies are rendered in the "More Portfolio Companies" section on the frontend, separate from scraped listings.

Each job gets a stable 12-character ID derived from an MD5 hash of its URL. A `seen` set prevents duplicates within a single run in case the same job URL appears more than once.

### Platform adapters

Most platforms expose a proper machine-readable endpoint: Greenhouse's REST API, Lever's JSON API, TeamTailor's RSS feed, Workable's Markdown endpoint, and YC's embedded page JSON. These adapters are straightforward HTTP GET calls followed by parsing.

The Notion adapters (`notion_collection`, `notion_toggles`) call Notion's internal (undocumented) APIs: `queryCollection` for database-style boards and `loadPageChunk` for toggle-block pages. No Notion API key is required for public pages. These adapters are more complex because Notion's internal API format is inconsistent across page types.

The two custom HTML adapters (`zime_html`, `devicethread_html`) parse raw HTML with regex. They are the most fragile part of the system: any change to the careers page markup will break them.

### Frontend filtering (`components/JobBoard.tsx`)

`inferFunction()` classifies each job into a functional area (Engineering, Product, Sales & Success, Marketing, Operations, HR, Design, Other) by running regex against the combined `title + department` string. The order of checks is load-bearing: HR is tested before Engineering so "Technical Recruiter" is not caught by the Engineering patterns; Marketing is tested before Sales so PMM roles are not caught by the GTM keyword in the Sales pattern.

`normalizeRegion()` maps raw location strings to four regions. Remote is checked first so a value like "India, Remote" maps to Remote rather than India.

The `page.tsx` Server Component reads `data/jobs.json` and `data/companies.json` from disk at build time (using Node's `readFileSync`). This means the board shows the state of the JSON at the last Vercel deploy, not live data. Job listings update only when the scraper runs and Vercel rebuilds.

### Tech stack

| Dependency | Version | Purpose |
|---|---|---|
| Next.js | 16.2.6 | React framework; static file reads at build time |
| React | 19.2.4 | UI rendering |
| Tailwind CSS | 4 | Utility-first styling |
| TypeScript | 5 | Type safety for frontend components |
| Python | 3.11 | Scraper runtime |
| requests | latest | HTTP calls to ATS APIs |
| GitHub Actions | — | Weekly scheduling, auto-commit of `data/jobs.json` |
| Vercel | — | Hosting and automatic redeploy on push |

### Limitations

**Static data at build time.** The frontend reads `data/jobs.json` when Vercel builds the site. Jobs added by a company between Monday scrapes are invisible until the next run.

**No job history.** `data/jobs.json` is overwritten on every run. There is no record of when a job was first posted or removed.

**Custom HTML scrapers are fragile.** The `zime_html` and `devicethread_html` adapters rely on regex patterns tied to current markup. A careers page redesign will silently return 0 jobs with no error.

**Notion IDs change.** The `notion_collection_id`, `notion_view_id`, and `notion_page_id` values come from Notion's internal API. Recreating or duplicating a Notion page generates new IDs, requiring a manual update to `companies.json`.

**No retry logic.** A single HTTP timeout or non-200 response causes that company's scrape to log an error and continue. That company's jobs are absent from `data/jobs.json` until the next successful run.

**Notion authentication.** The Notion adapters only work for public Notion pages. A company making their careers page private will cause that adapter to fail.

**No rate limiting.** The scraper fires all requests sequentially but without delays. Platforms with strict rate limits may start returning errors if the company count grows significantly.
