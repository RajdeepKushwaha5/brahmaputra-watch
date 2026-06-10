# Brahmaputra Watch

The last mile for a flood forecast that already exists.

India's Central Water Commission publishes flood forecasts with up to 24 hours
of lead time at [ffs.india-water.gov.in](https://ffs.india-water.gov.in/) — on a
dashboard, in English, in cumecs. Assam's floods affect millions of people every
year ([2.42M+ in 2025](https://ddnews.gov.in/en/assam-floods-52-people-dead-over-24-lakh-affected-as-crisis-deepens/),
[2.5M flood displacements per IDMC GRID 2025](https://www.internal-displacement.org/spotlights/india-assam-floods-highlight-the-need-to-reduce-displacement-risk/)).
This project turns a public river-discharge forecast into a plain-language,
bilingual Telegram alert for one village's coordinates, orchestrated by
[Kestra](https://kestra.io).

## Architecture

Five flows (plus the naive v1, kept for the write-up):

| Flow | Trigger | Job |
|---|---|---|
| `calibrate_river_baseline` | Weekly (Mon 03:00) | 40 years of GloFAS reanalysis → monsoon-season p90/p98 → KV store |
| `monitor_river` | Every 6 h | Fetch 7-day ensemble forecast, classify NORMAL/WATCH/DANGER, alert **only on transitions** |
| `send_alert` | Subflow | Gemini rewrites the bulletin in Assamese + English; `errors:` branch sends a plain numeric fallback |
| `escalate_severe` | Subflow (DANGER only) | 15-min human review window via `Pause`; **fails open** to the district group on timeout |
| `watchdog` | Flow trigger on `FAILED` | Tells the operator when the pipeline itself breaks (lives in `assam.ops`, outside the watched namespace) |

Design rules the flows encode:

- **Calibrate, don't guess.** Danger thresholds come from the river's own
  1984–present monsoon distribution, not a number borrowed from Wikipedia.
- **Alert on transitions, not levels.** KV-store memory; severity changes fire
  messages, steady states don't. De-escalations alert too ("water receding").
- **Ensemble max, not median.** For warnings, a false negative costs more than
  a false positive.
- **The LLM translates; the orchestrator judges.** Severity is decided in
  deterministic YAML before the AI sees anything.
- **Fail open where silence kills.** AI down → plain numeric alert. Human
  reviewer absent → escalation goes out anyway, labelled unreviewed.

## Data sources

- [Open-Meteo Flood API](https://open-meteo.com/en/docs/flood-api) — GloFAS v4
  river discharge, ~5 km grid, free, **no API key**. Reanalysis from 1984,
  forecasts with ensemble aggregates (`river_discharge_max` etc.).
  - Forecast: `https://flood-api.open-meteo.com/v1/flood?latitude=26.02&longitude=89.98&daily=river_discharge_max&forecast_days=7`
  - History: same endpoint with `daily=river_discharge&start_date=1984-01-01&end_date=<today>`
- CWC station stage levels (manual cross-check): https://ffs.india-water.gov.in/

Default coordinates: **Brahmaputra at Dhubri (26.02, 89.98)** — the district
where [775,721 people were affected in 2025](https://www.insightsonindia.com/2025/06/03/assam/).
Change `latitude`/`longitude` inputs to watch any river GloFAS covers.

## Setup

1. **Run Kestra** (Docker):
   ```powershell
   Invoke-WebRequest -Uri "https://raw.githubusercontent.com/kestra-io/kestra/develop/docker-compose.yml" -OutFile "docker-compose.yml"
   docker compose up -d
   ```
   Open http://localhost:8080. The flows need `plugin-notifications` (Telegram),
   `plugin-ai`, and `plugin-script-python`. If a task type is reported missing,
   use the full image (`kestra/kestra:latest-full`) or install the plugin from
   the in-app Plugins page.

2. **Secrets** — base64-encode each value and add to the `kestra` service
   environment in `docker-compose.yml`, then `docker compose up -d` again:
   ```yaml
   environment:
     SECRET_TELEGRAM_BOT_TOKEN: <base64 of bot token from @BotFather>
     SECRET_GEMINI_API_KEY: <base64 of key from aistudio.google.com>
   ```
   PowerShell one-liner to encode:
   `[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("my-value"))`

3. **Telegram** — create a bot with [@BotFather](https://t.me/BotFather), add it
   to your group(s), get the chat id (e.g. via `https://api.telegram.org/bot<TOKEN>/getUpdates`
   after posting in the group). Put the chat ids in the `variables:` blocks of
   `send_alert.yml`, `escalate_severe.yml`, `watchdog.yml` (and v1).

4. **Import flows** — paste each file from `flows/v2/` (and `flows/v1/` if you
   want the cried-wolf baseline for comparison) into Kestra's editor, or use the API:
   ```powershell
   Get-ChildItem flows\v2\*.yml | ForEach-Object {
     Invoke-RestMethod -Uri "http://localhost:8080/api/v1/flows" -Method Post `
       -ContentType "application/x-yaml" -Body (Get-Content $_ -Raw)
   }
   ```

5. **Run order**:
   1. Execute `calibrate_river_baseline` once manually (seeds `WATCH_LEVEL` /
      `DANGER_LEVEL` in the KV store — `monitor_river` fails loudly without them,
      and the watchdog will tell you so; that's by design).
   2. `monitor_river` then runs on its schedule. Execute it manually to test.

## Testing the failure paths

- **Force a DANGER transition:** in the KV store UI, set `WATCH_LEVEL`/`DANGER_LEVEL`
  to values below the current forecast, delete `LAST_SEVERITY`, run `monitor_river`.
- **AI fallback:** temporarily break `GEMINI_API_KEY` → `send_alert` should still
  deliver the plain numeric message via the `errors:` branch.
- **Fail-open escalation:** trigger DANGER and don't resume the Pause → after
  15 min the unreviewed district alert must go out.
- **Watchdog:** delete the KV thresholds and run `monitor_river` → expect a
  Telegram message from `assam.ops.watchdog`.
