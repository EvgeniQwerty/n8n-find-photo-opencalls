# n8n Photography Open Calls Automation

Importable n8n workflows for finding current, free photography open calls with online submission, extracting structured details with Gemini, and saving them to Airtable.

This repository contains:

- `setup-airtable.json` - run once to create or validate the Airtable schema.
- `daily-photo-open-calls.json` - run manually once, then activate for daily 08:00 Asia/Ho_Chi_Minh discovery.

## How It Works

The automation is source-first to avoid burning search API quotas.

1. **Airtable setup**
   - `setup-airtable.json` creates or validates two tables:
     - `Sources` - known sites, aggregators, museums, galleries, festivals, and discovery metadata.
     - `Open Calls` - individual open call records with deadline, summary, fee status, confidence, and review status.

2. **Daily source recheck**
   - The daily workflow reads `Approved` and `Candidate` rows from `Sources`.
   - It fetches those source pages directly over HTTP.
   - It extracts likely open-call links from each page using anchor text and URL keywords such as `open call`, `call for entry`, `submission`, `deadline`, `photography`, `contest`, `award`, and `grant`.
   - This path does not spend search API credits.

3. **Periodic discovery**
   - Discovery is controlled by `DISCOVERY_MODE`.
   - Default `weekly` mode runs search-provider discovery once a week and also runs discovery when `Sources` is empty, so the first run can seed the database.
   - `daily` runs discovery every day.
   - `manual` disables search-provider discovery and only checks existing Airtable sources.

4. **Search provider fallback**
   - Brave Search is attempted first when `BRAVE_SEARCH_API_KEY` is present.
   - If Brave returns fewer than `MIN_RESULTS_BEFORE_FALLBACK`, SerpAPI is tried when `SERPAPI_API_KEY` is present.
   - If results are still below the threshold, Tavily Search is tried when `TAVILY_API_KEY` is present.
   - Tavily Extract is intentionally not used; page content is fetched directly over HTTP to preserve Tavily quota.

5. **AI extraction**
   - Gemini classifies each candidate URL and returns structured JSON.
   - The workflow asks for:
     - whether it is a photography/photo/video/multimedia open call;
     - whether required submission fee is absent;
     - whether online/email/platform submission is available;
     - deadline;
     - short summary;
     - confidence and rejection reason.

6. **Airtable output**
   - Confirmed current records become `Active`.
   - Unclear records, AI failures, rate limits, or exhausted run budgets become `Needs review`.
   - Past-deadline records become `Expired`.
   - Manual fields are preserved: `My Decision`, `Notes`, and `Priority`.

## API Roles

- **Airtable API**
  - Stores `Sources` and `Open Calls`.
  - Setup workflow uses Airtable Metadata API to create missing tables/fields.
  - Daily workflow uses Airtable Records API for reads and upserts.

- **Gemini API**
  - Performs structured extraction/classification.
  - Uses Gemini `generateContent` with JSON response schema.
  - Controlled by `GEMINI_MODEL`, `AI_MAX_REQUESTS_PER_RUN`, and `AI_TIME_BUDGET_MS`.

- **Brave Search API**
  - Primary search provider for periodic discovery if configured.
  - Good for broad web discovery without using Tavily credits.

- **SerpAPI**
  - Optional fallback search provider.
  - Useful when Brave is not connected or returns too few results.

- **Tavily Search**
  - Optional final fallback only.
  - Tavily Extract is not used.
  - Best kept as a backup because free monthly credits can disappear quickly during testing.

## Configuration

You can paste these directly into the config nodes after import:

- In `setup-airtable.json`, open the `Airtable Setup Config` node.
- In `daily-photo-open-calls.json`, open the `Open Calls Config` node.

Replace the `PASTE_..._HERE` placeholders with real values.

The workflows no longer read `process.env`, because n8n Code nodes may not expose `process`.

Alternatively, advanced users can move these values to n8n credentials or environment variables and adjust the config nodes.

Required values:

- `AIRTABLE_API_KEY` - Airtable personal access token. Paste it into the config node field named `AIRTABLE_API_KEY`.
- `AIRTABLE_BASE_ID` - target Airtable base id.
- `GEMINI_API_KEY` - Google AI Studio / Gemini API key for structured extraction.

Optional:

- `BRAVE_SEARCH_API_KEY` - primary search provider for periodic discovery.
- `SERPAPI_API_KEY` - optional fallback search provider.
- `TAVILY_API_KEY` - optional final fallback search provider. Tavily Extract is not used.
- `GEMINI_MODEL` - default `gemini-3.1-flash-lite`; alternatives: `gemini-2.5-flash-lite`, `gemini-2.5-flash`.
- `OPEN_CALL_CONFIDENCE_THRESHOLD` - default `0.75`.
- `OPEN_CALL_RESULTS_PER_QUERY` - default `8`.
- `DISCOVERY_MODE` - default `weekly`; use `daily` for more discovery or `manual` to disable search-provider discovery.
- `DISCOVERY_QUERY_LIMIT` - default `3`; recommended range `1-3`.
- `MIN_RESULTS_BEFORE_FALLBACK` - default `3`; use fallback provider only when the previous provider returns fewer results.
- `SOURCE_RECHECK_LIMIT` - default `10`; number of existing Airtable sources to check directly per run.
- `DIRECT_LINK_LIMIT_PER_SOURCE` - default `8`; max candidate links extracted from each source page.
- `OPEN_CALL_MAX_CANDIDATES` - default `5`; recommended range `3-20`.
- `AI_MAX_REQUESTS_PER_RUN` - default `8`; recommended range `3-30`.
- `AI_TIME_BUDGET_MS` - default `240000`, so the Code node stops before n8n's 300 second task timeout.

Daily runs first recheck existing Airtable `Sources` with direct HTTP, which does not spend search API credits. Search-provider discovery runs only when `DISCOVERY_MODE` allows it. With the default `weekly` mode, discovery also runs when the `Sources` table is empty so the first run can seed the registry.

## Suggested Settings

Conservative weekly mode:

```text
DISCOVERY_MODE = weekly
DISCOVERY_QUERY_LIMIT = 2
OPEN_CALL_RESULTS_PER_QUERY = 10
MIN_RESULTS_BEFORE_FALLBACK = 3
SOURCE_RECHECK_LIMIT = 20
DIRECT_LINK_LIMIT_PER_SOURCE = 10
OPEN_CALL_MAX_CANDIDATES = 10
AI_MAX_REQUESTS_PER_RUN = 10
```

One-off wider discovery run:

```text
DISCOVERY_MODE = daily
DISCOVERY_QUERY_LIMIT = 4
OPEN_CALL_RESULTS_PER_QUERY = 10
OPEN_CALL_MAX_CANDIDATES = 20
AI_MAX_REQUESTS_PER_RUN = 20
```

After a one-off wider run, switch `DISCOVERY_MODE` back to `weekly`.

To avoid Tavily spend, leave `TAVILY_API_KEY` empty or keep `MIN_RESULTS_BEFORE_FALLBACK` low enough that SerpAPI results usually satisfy the fallback threshold.

## Usage

1. Import `setup-airtable.json` into n8n.
2. Run it once. It creates or validates the `Sources` and `Open Calls` tables.
3. Import `daily-photo-open-calls.json`.
4. Run it manually once and inspect the execution summary.
5. Activate the workflow.
