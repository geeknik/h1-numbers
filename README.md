# h1-numbers

Local dashboard that runs analytics on your HackerOne data.

Pulls from the [HackerOne hacker API](https://api.hackerone.com/getting-started-hacker-api/),
stores everything in SQLite, and serves a small FastAPI dashboard at
`http://127.0.0.1:8765`.

## What it tracks

- **You** — your reports, earnings, payouts, severity/state breakdowns,
  monthly trends, top-paying programs, top weaknesses, recent reports.
- **Public hacktivity** — disclosed-report sample with severity mix,
  top paying programs, top CWEs, top reporters, monthly trends.
- **Programs** — overview of accessible programs, bounty/open-scope counts,
  submission-state distribution.

## Setup

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Configure `.env` (already created — confirm your username is correct):

```
HACKERONE_API_USERNAME=your_h1_username
HACKERONE_API_KEY=your_api_token
```

Get an API token at <https://hackerone.com/users/edit/api_tokens>.

## Usage

Fetch all your data into SQLite (first run takes a minute or two):

```bash
python -m h1n.cli fetch
```

Or fetch a single scope:

```bash
python -m h1n.cli fetch --scope my_reports,earnings
```

Available scopes: `my_reports`, `earnings`, `payouts`, `programs`, `hacktivity`.

Run the dashboard:

```bash
python -m h1n.cli serve
```

The server prints a refresh token at startup — paste it into the dashboard's
refresh-token input to trigger refetches from the UI. (Localhost-only by
default; the token prevents drive-by CSRF from anything running locally.)

Open <http://127.0.0.1:8765/>.

## Architecture

```
h1n/
├── client.py     httpx Basic-auth client, paginated GET, 429 backoff
├── config.py     env loader (HACKERONE_API_*, H1N_*)
├── db.py         SQLite schema + upserts, raw_json preserved
├── fetcher.py    JSON:API → flat rows, scope-by-scope ETL
├── analytics.py  read-only SQL queries
├── server.py     FastAPI routes + background refresh
└── cli.py        `python -m h1n.cli {fetch,serve,status}`

frontend/
├── index.html    tabs: My reports / Hacktivity / Programs
├── app.js        Chart.js charts, table rendering, refresh trigger
└── styles.css    dark theme

data/h1.db        SQLite (auto-created, gitignored)
```

## Hardening notes

- `.env` is gitignored. Token is loaded once via `python-dotenv`,
  passed only to `httpx.BasicAuth`, never logged.
- `client.py` strips error details so credentials never end up in logs.
- Server binds `127.0.0.1` by default. `/api/refresh` requires a
  per-process token (printed at startup, hashed-compared) to prevent
  any local browser from triggering an unauthenticated fetch.
- All SQL uses parameterized queries; user-controlled `limit` values
  are clamped before formatting.
- `H1N_HACKTIVITY_MAX_PAGES` and `H1N_PROGRAMS_MAX_PAGES` bound
  ingestion size — hacktivity is huge; you want a sample, not the firehose.
- Pagination is bounded at the client; pages are clamped to ≤100 items
  (HackerOne's max), and a small sleep between pages stays well below
  the 600 reads/min limit.

## Re-fetching

Each `fetch` upserts by id, so subsequent runs are incremental updates.
A scheduled cron / launchd job hitting `python -m h1n.cli fetch` daily
will keep the dashboard fresh.

### Hacktivity specifics

The hacktivity API has two hard limits:

- Page size caps at ~50 (we ask for 100 anyway).
- Pagination depth caps at <100 pages — a single linear walk reaches
  ~5000 newest disclosures, which is roughly mid-2017 to today.

The fetcher mitigates both:

- **Watermark.** On every non-`--full` run, `fetch_hacktivity` reads
  `MAX(disclosed_at)` from SQLite and adds `disclosed_at:>=YYYY-MM-DD`
  (with one day of overlap) to the Lucene query, so daily refreshes
  only pull what's new.
- **`--full` flag.** Bypasses the watermark and re-paginates from the
  newest disclosure to the API depth cap. Use after a long pause or
  to verify completeness:

  ```bash
  python -m h1n.cli fetch --scope hacktivity --full --hacktivity-max-pages 99
  ```

  Takes ~80s for ~4950 items.

The dashboard's refresh button has the same `full` toggle.

## License

[Apache License 2.0](LICENSE)
