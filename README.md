# Rank Tracker

A self-hosted SEO rank tracker built on the [Decodo SERP Scraping API](https://decodo.com/scraping/serp). Tracks organic position, AI Overview citations, featured snippets, and PAA presence across keywords, locations, and devices. Stores everything in SQLite and prints trend reports from the terminal.

## Quick start

```bash
# 1. Create a virtualenv and install
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 2. Provide credentials
cp .env.example .env
# Edit .env and paste the "Basic authentication token" from your Decodo dashboard
# (Web Scraping API → API Playground → top-right auth field)

# 3. Smoke-test the setup
python main.py doctor

# 4. Edit keywords.csv to your domains, then run a check
python main.py check
```

## Commands

| Command | What it does |
|---|---|
| `python main.py doctor` | Verifies `DECODO_AUTH`, DB writability, live Decodo API, **last-run freshness**, and lifetime API call count. `--skip-api` skips the network call. `--max-staleness-hours N` warns if no successful run in N hours (default 48). |
| `python main.py test-alert` | Sends a sample alert through every configured channel and reports per-channel HTTP status. Use this to validate webhook setup before relying on alerts. |
| `python main.py check` | Runs one batch over `keywords.csv`. Flags: `--concurrency N` (parallel fetches), `--alerts` (send threshold-triggered alerts), `--threshold N` (alert sensitivity in positions, default 3), `--delay 1.0` (seconds between sequential calls). Logs API-call count at end of run. |
| `python main.py schedule --every 86400` | Runs `check` on a fixed interval. Foreground — use cron / launchd / systemd (see `examples/`) for unattended scheduling. |
| `python main.py report` | Latest snapshot per (keyword × geo × device × locale) with visibility score and Δ vs prior run. Add `--keyword "..."` for a chronological trend, `--location "..."` to filter, or **`--html report.html`** for a self-contained shareable HTML page. |
| `python main.py competitors --keyword "..."` | **Top-N organic results for a keyword + diff vs prior run** (who entered, who dropped, who moved). Filter with `--location`, `--device`, `--locale`. |
| `python main.py history` | Raw chronological dump. `--keyword "..."` to filter. |
| `python main.py export --format csv --output results.csv` | Export full DB or one run (`--run-id N`) to CSV or JSON. |

All commands accept `--db path/to.db` and `--log-level DEBUG`.

## keywords.csv

Required columns: `keyword`, `target_domain`. Optional: `geo`, `locale`, `device_type`, `google_results_language`, `pages`, `target_url`, **`active`**, **`frequency`**.

```csv
keyword,target_domain,geo,locale,device_type,google_results_language,pages,active,frequency
best laptop 2026,wired.com,United States,en-us,desktop,en,1,yes,daily
serp scraping api,decodo.com,United States,en-us,desktop,en,1,yes,weekly
residential proxies,decodo.com,United States,en-us,desktop,en,1,yes,monthly
old-campaign-keyword,decodo.com,United States,en-us,desktop,en,1,no,daily
```

- `target_domain` matches subdomain-aware (e.g. `example.com` matches `https://blog.example.com/...` but not `https://example.com.evil.com/...`).
- `target_url` (optional) matches the exact URL instead — useful for tracking one specific blog post.
- `geo` accepts country names, "City,Region,Country", ISO codes, or coordinates. See Decodo's [geolocation docs](https://help.decodo.com/docs/web-scraping-api-google-geolocation).
- **`active=no`** keeps the historical data but stops new checks — pause without forgetting.
- **`frequency`** is one of `daily` (~22h window), `weekly` (~6.5d), `monthly` (~28d), or `paused`. Each row is skipped if the same (keyword × geo × device × locale) was checked within the window. Run the script as often as your cron fires — frequency rules guard against double-billing.
- Each row is one check. To track a keyword across three locations, write three rows. **Exact duplicates are auto-collapsed at load time** with a warning.

### Cost control

The combination of `active` + `frequency` is the practical cost lever. On a 50-keyword list with a mix of daily/weekly/monthly tiers, expect 30–60% fewer Decodo calls than tracking every keyword daily.

API call count is logged at the end of every run and surfaced in `python main.py doctor`.

## Visibility score

A single number per result that blends position with SERP-feature ownership. Used in the `report` summary and available via `scoring.visibility_score()` if you want to alert on it instead of raw position.

```
score = 100 / √position
      + 30 if featured_snippet_owned
      + 25 / √citation_rank if ai_overview_cited
      + 3 if paa_present
```

Position 1 baseline = 100. AI citation at rank 1 = +25. So `pos 2 + AI-cited #1 + PAA` ≈ 98.7.

## Architecture

| File | Role |
|---|---|
| `models.py` | Pydantic `RankCheckConfig`, `RankResult`, `AIOverviewCitation`, `CompetitorEntry`. |
| `fetcher.py` | Decodo client with retries, distinct `DecodoCredentialError` / `DecodoAPIError`, and the response parser. |
| `canary.py` | Schema-drift validator. Asserts structural shape (path, types, key presence) without false-alarming on content-level absence (e.g. a query that legitimately lacks PAA). |
| `storage.py` | SQLite layer with WAL mode + migrations. `runs` + `rank_results` tables. Per-call connections — thread-safe. |
| `scheduler.py` | `BatchRunner` (sequential or `ThreadPoolExecutor`), keyword loader, frequency-tier filter, `Alerter` (Slack/Discord/SMTP), heartbeat. |
| `scoring.py` | Composite visibility score. |
| `report.py` | Terminal summary + trend + competitor diff + self-contained HTML report. |
| `main.py` | CLI: `doctor`, `check`, `schedule`, `report`, `competitors`, `export`, `history`, `prune`, `test-alert`. |
| `keywords.csv` | Sample input. |
| `tests/test_all.py` | 109 unit + integration tests against a captured fixture (no network needed). |
| `examples/` | cron / launchd / systemd templates for unattended runs. |

## Alerting

Set any combination of these in `.env` and add `--alerts` to `check` or `schedule`:

```
ALERT_WEBHOOK_URL=https://hooks.slack.com/services/...   # or Discord
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=you@example.com
SMTP_PASSWORD=app-password
ALERT_EMAIL_FROM=you@example.com
ALERT_EMAIL_TO=team@example.com
```

Webhook payload is selected automatically by URL:
- `hooks.slack.com/...` → `{"text": "..."}` (Slack)
- `discord.com/api/webhooks/...` or `discordapp.com/...` → `{"content": "..."}` (Discord)
- Anything else → `{"text": "..."}` (generic)

Microsoft Teams uses a different webhook protocol (Power Automate with Adaptive Card payloads since the 2024 deprecation of the legacy O365 Connector). Treat Teams as a separate integration; the simple `{"text": ...}` payload won't work.

### Heartbeat after every run

When `--alerts` is enabled, a per-run summary is posted to the same channel after each `check` / `schedule` run, even if no thresholds were breached:

```
Rank tracker heartbeat — run #12
• 8 check(s) saved; 8 API call(s) this run (lifetime 96).
• Top movers (≥3 positions):
   - 'best laptop 2026' (United States/desktop): 1→4 ↓
```

Silence in your Slack now means the tracker stopped — not "everything's fine."

If no channel is configured, alerts are logged to stderr (no silent drops). If a webhook returns 4xx/5xx, the response status and body are logged as a warning (no silent drops either).

**Before relying on alerts**, verify delivery:

```bash
python main.py test-alert
# or with a custom message:
python main.py test-alert --message "Hello from rank tracker"
```

Exit codes: 0 = all channels delivered, 1 = one or more failed, 2 = nothing configured.

Alert events:
- `NEW` — keyword entered the SERP for the first time at this (location, device).
- `IMPROVED` / `DROPPED` — position moved by ≥ `--threshold` slots.
- `DROPPED OUT` — was ranked, now isn't.

## Running tests

```bash
source .venv/bin/activate
python -m unittest tests.test_all -v
```

The suite uses a captured real Decodo response (`tests/fixtures/`) plus `unittest.mock` for transport — fully offline. Covers credential errors, retries on 5xx, 401 short-circuit, the parser (organic position, AI Overview citation, PAA, exact-URL match, dimensions), domain matching edge cases (substring traps, www stripping, subdomains), SQLite round-trip, alerter branches, CSV loader edge cases, and the BatchRunner (sequential, concurrent, one-failure-doesn't-abort).

## Exit codes

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | `doctor` reported at least one failure |
| 2 | No keyword configs found in CSV |
| 3 | Export wrote 0 rows |
| 4 | Decodo rejected credentials |
| 5 | Required file not found |
| 130 | Interrupted (Ctrl+C) |

## Unattended scheduling

`schedule` runs in the foreground. For real automated runs, use your OS scheduler — templates included:

| Platform | Template | Install |
|---|---|---|
| Linux/macOS cron | [examples/crontab.example](examples/crontab.example) | `crontab -e`, paste the line, save |
| macOS launchd | [examples/com.local.rank-tracker.plist](examples/com.local.rank-tracker.plist) | `cp ... ~/Library/LaunchAgents/ && launchctl load ...` |
| Linux systemd (user) | [examples/rank-tracker.service](examples/rank-tracker.service) + [examples/rank-tracker.timer](examples/rank-tracker.timer) | `cp ... ~/.config/systemd/user/ && systemctl --user enable --now rank-tracker.timer` |

After installing, `python main.py doctor` will warn if the scheduled job stops landing (no successful run in 48h by default).

## Cost visibility

Each call to Decodo counts. `BatchRunner` records `api_calls` per run (including retries), and `doctor` prints the lifetime total. After a run you'll see:

```
Run 7 complete — 8 API call(s) this run, 142 lifetime
```

For 50 keywords × 3 locations × daily, expect ~4,500 calls/month.

## Operational hygiene

### Database retention

The DB grows by one row per (keyword × geo × device) per run. After a year of daily runs on 50 keywords that's ~18k rows — fine — but you should still prune periodically:

```bash
# Dry-run (prints what would be deleted, deletes nothing):
python main.py prune --older-than-days 365

# Actually do it (also runs VACUUM to reclaim disk space):
python main.py prune --older-than-days 365 --yes
```

Pair with a monthly cron entry. The schema uses `ON DELETE CASCADE`, so dropping old `runs` removes their `rank_results` automatically.

### Backups

The whole database is one file (`rank_tracker.db`). Back it up with SQLite's online backup:

```bash
sqlite3 rank_tracker.db ".backup '/path/to/backups/rank_tracker-$(date +%F).db'"
```

A weekly cron line is enough — the file compresses well too.

### Log rotation

If you redirect `python main.py check` output to a log file in cron (`>> rank-tracker.log 2>&1`), use `logrotate` so it doesn't grow unbounded:

```
/path/to/rank-tracker.log {
    weekly
    rotate 8
    compress
    missingok
    notifempty
}
```

## Scaling beyond SQLite

The brief calls out the migration triggers: concurrent writes from multiple processes, multiple operators, or tens of thousands of results per day. For those scales, swap `Storage` for a Postgres-backed implementation with the same method surface — every other module talks to `Storage` through that interface only.
