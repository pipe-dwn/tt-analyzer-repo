# Trade Journal: Project Brief

## Purpose
Private, read-only web app that auto-logs my Tastytrade trades (mainly 2-4 DTE
credit spreads) and explains WHY each trade performed as it did. It never places
trades and never feeds entry decisions.

## Stack (keep it lean)
Cloudflare Workers (cron + API) + D1 (SQLite) + static frontend, deployed from
GitHub. Cloudflare Access protects the site (email one-time code). No other
vendors. TypeScript throughout.

## Hard rules
- Read-only Tastytrade scope. Secrets only in .dev.vars (gitignored) and
  Cloudflare secrets. Never commit secrets, never expose them to the browser.
- Sandbox vs production selected by one env variable. Sandbox REST host:
  api.cert.tastyworks.com; production: api.tastyworks.com. Verify endpoints and
  auth against developer.tastytrade.com before coding; do not rely on memory.
- Store all timestamps in UTC; display in America/Chicago. Parse ISO 8601 only.
- If P&L disagrees with Tastytrade's own numbers, Tastytrade is right.
- Facts and hypotheses are always separate categories:
  MEASURED (entry conditions, greeks, IV/IVR, excursion metrics, price path vs
  strikes) vs CONTEXTUAL (macro, news, sector; each labeled a hypothesis with a
  source). "No identifiable catalyst" is a valid output.

## Data notes
- Underlying history: DXLink Candle events (symbol like SPY{=5m} plus fromTime).
- Known trap: some libraries hardcode "contract": "AUTO" on the candle channel
  and get empty history. Use "contract": "HISTORY" for backfill.
- DXLink publishes greeks only live, so snapshots are the only source of
  greeks/IV over time.
- Archive each trade's candle window (underlying, SPY, VIX) into D1 at close.
- Cloudflare free tier limits subprequests per invocation; batch, and flag if
  we need the paid Workers plan.

## Reliability (top coding priority)
Must work with minimal bugs on Chrome, Edge, Safari (desktop + iPhone).
- Boring, widely supported web tech; one mature chart library.
- Mobile-first layout; wide tables scroll inside their own container; avoid 100vh.
- Every screen has loading, empty, and error states.
- Playwright tests on Chromium and WebKit with a mobile viewport before deploys.
- Display trade counts on every aggregate slice; dim buckets under ~30 trades.

## Screens
1. Home: sync health (last successful snapshot), open trades, week/month P&L,
   scrollable recent-trade feed.
2. Trade detail: header, candle chart with entry/exit markers and strike lines
   (SPY/VIX toggles), measured section, context section, summary with confidence
   line, my notes field.
3. Insights: win rate and P&L by ticker, IVR bucket, DTE, time of day, day of
   week, VIX regime, event-in-window; winners vs losers.
4. Status: sync history, errors, sandbox/production indicator.

## Milestones (do in order; stop and report after each)
0. Candle test: fetch underlying and option-leg candles for a real past spread;
   report what exists, depth, and gaps.
1. Snapshot Worker (~5 min, market hours, Central) + D1 schema.
2. Transaction sync and spread matching; verify P&L vs Tastytrade.
3. Dashboard.
4. AI summaries (Anthropic API + web search): sections are entry setup, path,
   backdrop, exit, confidence. Source links required; no causal claims that
   can't be time-aligned.
Deferred: iPhone Home Screen install.
