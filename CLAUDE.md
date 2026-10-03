# Trade Journal: Project Brief

## Purpose
Private, read-only web app that auto-logs my Tastytrade trades (mainly 2-4 DTE
credit spreads) and explains WHY each trade performed as it did. It never places
trades and never feeds entry decisions.

## Environment (IMPORTANT)
- No local environment. You work in a cloud sandbox and push to GitHub; Cloudflare
  Workers Builds auto-deploys from `main`. You cannot run Wrangler, reach
  Tastytrade, or see my secrets.
- I create the D1 database and set secrets in the Cloudflare dashboard. Tell me
  exact names and steps; never ask me to paste secrets into chat.
- Single Worker serves both API and static frontend (Workers static assets).
  Bindings and cron triggers are declared in the repo's Wrangler config.
- Schema changes: put SQL in /migrations; I run it in the D1 console. Tell me
  exactly what to paste and in what order.
- Debug loop: build a /debug page FIRST showing raw API responses, errors, and
  last-run status, so I can relay what happened. Verify with unit tests against
  recorded fixtures wherever possible, since you can't call live APIs.
- Never expose real trade data on a public URL: /debug and all data routes must
  sit behind Cloudflare Access before production credentials are added.

## Stack
Cloudflare Workers (cron + API) + D1 + static frontend. TypeScript. No other
vendors until the optional AI-summary milestone.

## Hard rules
- Read-only Tastytrade scope. Secrets only in Cloudflare; never in the repo or
  browser.
- Sandbox vs production via one env variable. Sandbox REST: api.cert.tastyworks.com;
  production: api.tastyworks.com. Verify auth and endpoints against
  developer.tastytrade.com before coding; do not rely on memory.
- Store UTC, display America/Chicago, parse ISO 8601 only.
- If P&L disagrees with Tastytrade's numbers, Tastytrade is right.
- MEASURED (entry conditions, greeks, IV/IVR, excursion metrics, path vs strikes)
  and CONTEXTUAL (macro, news; labeled hypotheses with sources) are always
  separate. "No identifiable catalyst" is a valid output.

## Data notes
- Underlying history: DXLink Candle events (e.g. SPY{=5m} plus fromTime).
- Known trap: some libraries hardcode "contract": "AUTO" on the candle channel and
  receive empty history. Use "contract": "HISTORY" for backfill.
- DXLink publishes greeks only live; snapshots are the only source of greeks/IV
  over time.
- Archive each trade's candle window (underlying, SPY, VIX) in D1 at close.
- Free tier limits subrequests per invocation; batch, and flag if the paid plan
  is needed.

## Reliability (top priority)
Minimal bugs on Chrome, Edge, Safari (desktop + iPhone).
- Boring, widely supported tech; one mature chart library.
- Mobile-first; wide tables scroll in their own container; avoid 100vh.
- Loading, empty, and error states on every screen.
- Playwright (Chromium + WebKit, mobile viewport) in CI if feasible; otherwise
  give me a short manual checklist per milestone.
- Show trade counts on every aggregate slice; dim buckets under ~30 trades.

## Screens
1. Home: sync health, open trades, week/month P&L, recent-trade feed.
2. Trade detail: candle chart with entry/exit markers and strike lines (SPY/VIX
   toggles), measured section, context section, summary with confidence line,
   my notes field.
3. Insights: win rate and P&L by ticker, IVR bucket, DTE, time of day, day of
   week, VIX regime, event-in-window; winners vs losers.
4. Status: sync history, errors, sandbox/production indicator.

## Milestones (in order; stop and report after each)
0. /debug/candles endpoint that fetches underlying and option-leg candles for a
   spread I specify, and prints what exists, depth, and gaps.
1. Snapshot Worker (~5 min, market hours, Central) + D1 schema.
2. Transaction sync and spread matching; I verify P&L against
