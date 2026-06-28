# RITA Slash Command Examples

Quick reference for every command. Use this to choose the right command and copy-paste a starting invocation.

---

## Daily Workflow

### `/start-day`
No arguments. Reads `PLAN_STATUS.md` and reports today's tasks + which command to use for each.

```
/start-day
```

---

### `/end-day`
No arguments. Runs all 4 steps: PLAN_STATUS → roadmap HTML → Confluence → git commit.

```
/end-day
```

---

## Bug Fixes

### `/fix-bug` — JS frontend bug, no server start

```
/fix-bug The Market Signals section loads blank — RSI and MACD KPIs all show —

/fix-bug FnO Positions table shows NaN for PnL column after instrument switch

/fix-bug Run Day button on Agent Panel does nothing when clicked

/fix-bug Trade Journal chart is empty but the table below it renders correctly
```

---

## Backend Engineering

### `/add-endpoint` — new or modified FastAPI route

```
/add-endpoint Add GET /api/v1/portfolio/greeks-summary that returns delta, gamma, theta, vega aggregated across all open positions

/add-endpoint Add POST /api/v1/alerts/dismiss that marks an alert as read by alert_id

/add-endpoint Modify GET /api/v1/market-signals to also return the trend_score_ma5 (5-day MA of trend_score)

/add-endpoint Add Experience Layer endpoint GET /api/experience/risk-snapshot that combines risk-timeline + performance-summary into one payload for the mobile home screen
```

---

### `/add-db-model` — new ORM model, repository, schema, migration

```
/add-db-model Add a WatchlistItem model — fields: item_id (PK), instrument (str), added_at (datetime), note (str nullable)

/add-db-model Add an AlertRule model — fields: rule_id (PK), rule_type (str), threshold (float), is_active (bool), created_at (datetime)

/add-db-model Add a DailySnapshot model to store EOD portfolio values — fields: snapshot_id, date, portfolio_value, benchmark_value, allocation_pct, regime
```

---

### `/add-chat-intent` — new classifier intent

```
/add-chat-intent Add intent: user asks about current Greeks exposure across the FnO portfolio (delta, gamma, theta, vega totals)

/add-chat-intent Add intent: user asks how much drawdown budget is remaining ("How much more can I lose?", "What is my drawdown headroom?")

/add-chat-intent Add intent: user asks to compare ASML vs NIFTY performance over the last month
```

---

## RITA Dashboard Features

### `/add-rita-feature` — new section, card, or chart in `rita.html`

```
/add-rita-feature Add a new "Watchlist" section (sec-watchlist) that shows a table of user-tracked instruments with their latest price and % change from GET /api/v1/watchlist

/add-rita-feature Add a "Signal Strength" KPI card to the Health strip showing the composite signal score (0–100) from the market-signals endpoint

/add-rita-feature Add a "Drawdown Budget" chart to the Risk section — a horizontal bar showing used vs remaining drawdown budget — reads from GET /api/v1/risk-timeline

/add-rita-feature Add a refresh button to the Performance section that re-calls loadPerformance() and updates all charts in place
```

---

## FnO Dashboard Features

### `/add-fno-feature` — new section, card, or chart in `fno.html`

```
/add-fno-feature Add a "Portfolio Greeks" summary card to the FnO overview showing total delta, gamma, theta, vega aggregated across all positions

/add-fno-feature Add a "Margin Utilisation" doughnut chart to the Margin section showing used vs available margin

/add-fno-feature Add an "Expiry Countdown" badge to each row in the Positions table showing days until expiry

/add-fno-feature Add a "P&L by Instrument" bar chart section that groups PnL from positions by underlying (NIFTY, BANKNIFTY, ASML, NVIDIA)
```

---

## Ops Dashboard Features

### `/add-ops-feature` — new section, card, or chart in `ops.html`

```
/add-ops-feature Add a "System Health" overview card showing uptime, last restart time, and DB connection status from GET /health

/add-ops-feature Add a "Recent Errors" table section that shows the last 10 API errors from GET /api/v1/metrics/errors

/add-ops-feature Add a "Data Freshness" indicator to the Overview section showing the date of the latest OHLCV row for each instrument

/add-ops-feature Add a "User Activity" table to the Users section showing last login time and request count per user
```

---

## Data Pipeline Features

### `/add-data-feature` — new indicator, analysis, or model

```
/add-data-feature Add a williams_r_14 indicator (Williams %R, 14-period) to the market signals pipeline — expose it via GET /api/v1/market-signals

/add-data-feature Add a momentum_score column to the OHLCV pipeline: composite of RSI, MACD histogram, and EMA crossover normalised to [0, 1]

/add-data-feature Add a rolling_correlation field that computes the 30-day rolling correlation between NIFTY and BANKNIFTY close prices — expose via a new GET /api/v1/correlation endpoint

/add-data-feature Retrain the NIFTY model with updated hyperparameters: learning_rate=0.0005, gamma=0.98, n_episodes=500 — write output to data/output/NIFTY/

/add-data-feature Add NVIDIA as a seeded instrument in market_data_cache — follow the same bulk-insert pattern used for ASML
```

---

## Mobile PWA Features

### `/add-mobile-feature` — new screen or update to `android-mobile-app/index.html`

```
/add-mobile-feature Add a new "Alerts" screen (s9) that fetches GET /api/v1/alerts and displays a list of recent alert cards with dismiss buttons

/add-mobile-feature Add a "Greeks Summary" card to the Portfolio screen (s8) showing total delta and theta from GET /api/experience/greeks-summary

/add-mobile-feature Add a "Watchlist" tab (tab 5) linking to a new screen showing tracked instruments and their current price from GET /api/v1/watchlist

/add-mobile-feature Update the Market screen (s3) to show williams_r_14 as a 5th KPI alongside the existing RSI, MACD, BB, ATR row
```

---

## General / Multi-step Tasks

### `/engineer-task` — use when no specialised command fits

```
/engineer-task Refactor the portfolio service to cache the last portfolio summary for 60 seconds using a module-level dict — avoid repeated DB reads on the hot path

/engineer-task Add request-id logging to all FastAPI routes — attach a UUID to the structlog context at the start of each request using a middleware

/engineer-task Move the paper positions seed data from main.py into a separate seed_paper_positions() function in a new file src/rita/seeds/paper_positions.py
```

---

## Chaining Commands

For tasks that touch multiple layers, chain commands explicitly:

```
# Add a new field end-to-end: data → API → UI
/add-data-feature Add williams_r_14 to the market signals pipeline
# (after agent confirms) →
/add-endpoint Expose williams_r_14 in GET /api/v1/market-signals response
# (after agent confirms) →
/add-rita-feature Add williams_r_14 KPI card to the Market Signals section
```

```
# New DB table end-to-end: model → endpoint → dashboard card
/add-db-model Add WatchlistItem model
# →
/add-endpoint Add GET /api/v1/watchlist and POST /api/v1/watchlist endpoints
# →
/add-rita-feature Add Watchlist section to rita.html
```
