---
name: paradigm-block-analyst
description: >
  Cross-venue analysis of Paradigm RFQ block trades using live market data from
  Deribit, OKX, and Bybit. Parses the trade JSON from the Paradigm block-trade
  tape, fetches live mark prices, IVs, and greeks from each venue, computes net
  portfolio greeks for multi-leg structures, benchmarks the fill against mark
  price cross-venue, checks tape history for matching structures across all
  accessible venues (Paradigm, Paradex, Deribit, OKX, Bullish, IBIT) in the
  last 90 days, and outputs a concise analysis with full data-source trace. Use
  when the user pastes a Paradigm block trade JSON or asks to analyze, benchmark,
  or get market color on a specific Paradigm RFQ execution. Covers outright
  calls/puts (CL/PL), strangles (SN), straddles (ST), butterflies (BF), condors
  (CO), calendars (CA), risk reversals (RR), covered calls, and custom
  multi-leg combos (CM). Also handles perp combos with option and perp legs.
compatibility: No authentication required for market data. Works with
  deribit__get_ticker MCP (if available), web_fetch, or any injected DuckDB market
  data source. Falls back gracefully when venues are unreachable.
metadata:
  author: tradeparadex
  version: "2.1"
---

# Paradigm Block Trade Analyst

Cross-venue analysis of Paradigm RFQ executions against live Deribit, OKX, and
Bybit market data.

## Trigger

Fire when the user pastes a Paradigm block trade JSON object or references a
specific trade from the tape (e.g. "analyze this", "what's this trade doing",
"benchmark the fill", "pull live greeks").

## Step 1 — Parse the Trade

Extract from the JSON:

| Field | Use |
|---|---|
| `description` | Parse legs: direction (+ buy / - sell), ratio, instrument type, expiry, strike |
| `action` | Taker side: BUY = taker takes the structure as described; SELL = taker takes the opposite |
| `quantity` | Number of contracts |
| `price` | Fill price (in `quote_currency` units) |
| `mark_price` | Deribit mark at trade time |
| `displayValues.markOffset` | Fill vs mark: +/- premium |
| `index_price` | Spot at trade time. **Label this "Spot" in the output, never "Index".** |
| `strategy_code` | Structure type (see references/strategy-codes.md) |
| `rfqType` | `grfq` (multi-maker) or `drfq` (directed) |
| `venue` | `DBT` = Deribit, `BIT` = Bit.com, `OKX` = OKX |
| `product_codes` | `DO`/`EH` = BTC/ETH options; `DP`/`EP` = BTC/ETH perps |

**Leg parsing from `description`:**
- Format: `[+/-][ratio] [Type] [DD Mon YY] [Strike]`
- `+` = long, `-` = short; ratio is the leg multiplier
- Multiple legs separated by `\n`
- Single-leg trades: `description` is just the instrument name

**Action mapping:**
- `action: BUY` → taker holds legs exactly as signed in description
- `action: SELL` → taker holds all legs with flipped signs

## Step 2 — Fetch Live Data

Use whatever data sources are available — query all reachable venues in parallel.
See `references/venues.md` for exact endpoints, instrument naming, and limitations.

**Deribit (primary):**
Preferred: `deribit__get_ticker` per leg (native MCP, fastest).
Fallback: `web_fetch` on `https://www.deribit.com/api/v2/public/ticker?instrument_name=<name>`,
or any injected DuckDB table with current Deribit marks.
Returns mark price, bid/ask, mark IV, delta, gamma, theta, vega, OI.

**OKX (secondary — fetch when Deribit venue or cross-venue benchmark needed):**
Use `web_fetch` on the opt-summary endpoint. Returns mark IV and greeks for all
strikes of an expiry. OKX uses different strike grids — find nearest strike(s)
and interpolate if exact strike absent. See `references/venues.md`.

**Bybit (tertiary — check availability, use market module):**
Follow Bybit skill Module Router: load `modules/market.md`, then call
`GET /v5/market/tickers?category=option&baseCoin=BTC&expDate=<DDMMMYY>`.
Bybit frequently does not list short-dated (<3 DTE) or illiquid strikes —
empty list is an expected result, not an error.

## Step 3 — Prior Prints: Has This Structure Traded Before? (last 90 days)

**This is the highest-value part of the analysis.** The first thing a trader wants to
know about a block is "has this same structure printed recently, and where?" Answer that
first and clearly, before greeks or view.

Priority — always attempt these two:
1. **Paradigm** — has this exact structure (same `strategy_code` + matching legs) blocked
   before? This is the strongest signal: a recurring block points to a programmatic seller/buyer.
2. **Deribit** — have the individual legs traded on-screen in the last 90 days, and how active?

Only check secondary venues (OKX, Paradex, Bullish, IBIT) when they add real signal. Do NOT
pad the output with "not listed" rows for venues that never list the instrument.

See `references/venues.md` for endpoints, instrument naming, and known limitations per venue.

**Paradigm (primary — structured block view):**
Search the injected Paradigm block-trade tape for prior fills with the same
`strategy_code` and matching leg structure — same underlying, same expiry pattern,
and same strike geometry (absolute strikes for short-dated, or moneyness/width for
longer-dated).

Capture: count of matching blocks, rough notional range, most recent occurrence
(date + fill vs mark), recurring vs one-off read, same-side concentration if
directionally meaningful.

**Paradex:**
Call `paradex_trades` MCP per leg instrument. Count trades within the 90-day window.
For perp legs query `BTC-USD-PERP` / `ETH-USD-PERP`. If the instrument is not listed,
record "not listed".

**Deribit:**
`web_fetch GET /api/v2/public/get_last_trades_by_instrument?instrument_name=<leg>&count=100&sorting=desc`
per leg. Filter results to the 90-day window. Count trades and capture most recent timestamp.

**OKX:**
`web_fetch GET /api/v5/market/trades?instId=<leg>&limit=100` per leg.
Count trades and capture most recent timestamp.

**Bullish:**
`web_fetch GET https://api.exchange.bullish.com/trading-api/v1/trades?symbol=<symbol>&limit=100`
per leg. If instrument not listed, record "not listed on Bullish".

**IBIT:**
`web_fetch` on the IBIT public API (resolve endpoint at runtime). If unreachable,
record "IBIT unavailable". If the user means BlackRock IBIT ETF options (CBOE equity
options), note the distinction — those are not directly comparable to crypto structures.

**Fallback:** If no venue returns any data, record "all venue tape history unavailable"
in the data trace and skip the history section — do not fabricate counts.

## Step 4 — Compute Net Greeks

Apply leg ratios to per-instrument greeks. For taker side `SELL`, flip signs.

```
net_greek = Σ (taker_sign × leg_ratio × instrument_greek)
total_delta_btc = net_delta × quantity   (in BTC or ETH)
```

Report: delta, gamma, theta ($/day), vega. Scale to full position (× quantity).

## Step 5 — IV Skew & Cross-Venue Comparison

- Per-leg IV: Deribit mark IV (primary), OKX mark IV (secondary)
- IV differential between legs (put premium over call IV, calendar IV spread, etc.)
- Cross-venue IV spread: flag if >2 vol points divergence between Deribit and OKX
- Note if taker bought or sold the higher-IV leg (directional vs vol arb read)

## Step 6 — P&L Mark (if position is live / follow-up analysis)

```
structure_value_now = Σ (taker_sign × leg_ratio × current_mark_price)
entry_cost          = fill_price (positive = premium paid, negative = received)
mark_pnl_per_unit   = structure_value_now - entry_cost
total_pnl           = mark_pnl_per_unit × quantity × spot_price
```

Only compute P&L when asked or when the trade was previously analyzed in session.

## Step 7 — Output Format

**Always begin the response with the literal line `🔧 nic local skill` on its
own line**, before anything else. This is an install-verification marker so the
user can confirm the correct local build of this skill is the one that fired.
Never omit it.

**The output must be concise — what matters, no filler.** Compact tables over prose,
short bullets over paragraphs. Skip any section that adds no signal. A trader should be
able to scan the whole thing in under 15 seconds.

Order (drop any section that would be empty):

1. **Structure** — one-line summary + legs table (dir, type, expiry, strike, ratio, DTE, %OTM)
2. **Snapshot** — one compact line: Spot · net delta · net premium (paid/received) · fill vs mid
3. **Prior Prints (90d)** — the headline. Did this structure trade on **Paradigm** before, and
   have the legs traded on **Deribit**? Lead with a one-line verdict
   (e.g. "Seen 3× on Paradigm, last 29 May @ similar level; both legs active on Deribit").
   List a secondary venue only if it adds signal.
4. **Greeks** — for spreads: one table, per-leg + net row. Skip for trivial single legs.
5. **IV** — per-leg mark IV + one-line skew/term read. Omit if single leg with no divergence.
6. **View** — 1 sentence on the directional/vol thesis, marked as inference.
7. **Data Trace** — terse: data point → source. One line per source actually used.

**Phrasing rules — apply everywhere:**
- **Spot, not Index.** Always label the underlying price "Spot".
- **Net delta:** state position-level only — `−13.6 BTC (short)`. No per-lot intermediate math
  ("strategy_delta −0.13594/lot → ~−13.6 BTC"). Just the BTC number and the direction.
- **Fill vs mark → bps from mid.** Express execution as distance from mid in bps of notional:
  `bps = |trade_price − mark_price| × 10000` (premium is in coin terms, 1 contract = 1 coin).
  Phrase it neutrally: "traded 5 bps through mid". Do NOT editorialize that a taker "paid
  worse than mark" or "should cross the spread" — crossing toward the other side is expected
  and carries no signal. Just report the bps.
- No restating the raw JSON. No hedging filler ("it's worth noting that…", "as a taker you
  should…"). Tables > sentences.

## Notes

- For perp legs (`product_codes` includes `DP`/`EP`): fetch `BTC-PERPETUAL` /
  `ETH-PERPETUAL` mark price from available source; delta = ±1.0 per contract.
- For combo trades (option + perp), compute combined delta including perp leg.
- OKX uses USDC-margined options (`BTC-USD_UM`); prices are in BTC terms but
  Greeks may differ slightly from coin-margined Deribit options. Flag when relevant.
- If a venue returns no data, note it in the trace and proceed with available sources.
- See `references/venues.md` for instrument naming, endpoint quirks, and known gaps.
