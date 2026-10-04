# Tastytrade MCP Connector: Project Brief

## Purpose
A private, read-only remote MCP server that lets me (in Claude chat) pull my
Tastytrade data on demand and analyze WHY my trades (mainly 2-4 DTE credit
spreads) performed as they did. It never places, modifies, or cancels orders.
No web frontend. Claude does the analysis; this server only supplies data.

## Environment (IMPORTANT)
- No local environment. You work in a cloud sandbox and push to GitHub;
  Cloudflare Workers Builds auto-deploys from `main`. You cannot run Wrangler,
  reach Tastytrade, or see my secrets.
- I create the Worker, KV namespace (OAUTH_KV), and secrets in the Cloudflare
  dashboard. Tell me exact names and steps; never ask me to paste secrets in chat.
- Bindings and any cron triggers are declared in the repo's Wrangler config.
- Since you can't call live APIs, verify with unit tests against recorded
  fixtures, and give me a /debug route (behind the same auth) that shows raw
  API responses, errors, and timestamps in copy-friendly form so I can relay
  results back to you.

## Stack
Cloudflare Worker, TypeScript, Cloudflare's remote MCP server pattern with
OAuth (workers-oauth-provider). Check Cloudflare's current docs and template
rather than relying on memory. No other vendors.

## Security (non-negotiable)
- Every MCP and debug route requires OAuth login via GitHub, restricted to an
  allowlist containing only my GitHub username. Deny by default. No
  unauthenticated route may return trade data, ever.
- Read-only Tastytrade scope. Expose NO tool that can trade or change anything.
- Secrets only in Cloudflare; never in the repo, logs, or tool output. Redact
  account numbers in outputs (show last 4 only).
- Sandbox vs production via one env variable. Sandbox REST: api.cert.tastyworks.com;
  production: api.tastyworks.com. Verify auth and endpoints against
  developer.tastytrade.com before coding.
- Do not enable production credentials until I confirm the allowlist login works.

## Tools to expose (read-only; concise, structured JSON; small default windows)
- get_transactions(start, end): raw fills, normalized.
- get_positions(): current open positions with marks.
- get_spreads(start, end): transactions matched into spreads (vertical credit
  spreads first; flag anything unmatched rather than guessing), with entry, exit,
  exit type (manual/expired/assigned/auto-closed), fills, and realized P&L.
- get_candles(symbol, interval, start, end): DXLink candles for underlyings and,
  if available, option legs. Report gaps and actual vs requested range.
- get_trade_context(spread_id): spread record + candles for the trade window for
  the underlying, SPY, and VIX + computed metrics (max adverse/favorable
  excursion in % and in expected moves, closest approach to short strike and
  when, time of largest move).
- get_chain_snapshot(symbol, expiration): current IV, greeks near strikes.
Keep output sizes bounded and paginated so tool results don't flood context.

## Data notes
- Candles: DXLink Candle events (symbol like SPY{=5m} plus fromTime).
- Known trap: some libraries hardcode "contract": "AUTO" on the candle channel
  and get empty history. Use "contract": "HISTORY" for backfill.
- DXLink publishes greeks only live. History of greeks/IV exists only if
  snapshots are stored (see Milestone 3).
- Free tier limits subrequests per invocation; batch, and flag if the paid plan
  is needed.
- Store/return timestamps in UTC with explicit offsets; document America/Chicago
  conversions. Parse ISO 8601 only.
- If P&L disagrees with Tastytrade's own numbers, Tastytrade is right.

## Analysis conventions (for the tool outputs, so Claude can follow them)
- Keep MEASURED data (entry conditions, excursion metrics, path vs strikes,
  greeks) clearly separate from CONTEXTUAL factors (macro, news), which Claude
  adds via web search as labeled hypotheses with sources.
- Every aggregate output includes trade counts; treat buckets under ~30 as hints.

## Milestones (in order; stop and report after each)
0. Auth skeleton + /debug route + sandbox connectivity. Then the candle test:
   fetch underlying and option-leg candles for a spread I specify and report
   what exists, depth, and gaps.
1. get_transactions, get_positions, get_spreads. I verify P&L against Tastytrade.
2. get_candles, get_trade_context, get_chain_snapshot.
3. Optional: D1 + cron snapshotting of open positions (greeks, IV, marks) and
   archiving of candle windows at close, exposed via new tools.
Deferred: any web frontend.
