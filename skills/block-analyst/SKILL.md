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
  version: "2.2"
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

## Step 3 — Prior Prints & Flow Impact (last 30 days)

**This is the highest-value part of the analysis. ALWAYS run the fetches below — never
report "not checked" or defer them as optional.** The trader's first questions are: has this
structure printed before, is one taker accumulating, and is the flow moving the market? Answer
concretely with counts, sizes, levels, and impact.

Two sources, both mandatory every time:

### 3a — Paradigm prior blocks (most important)
Block recurrence on Paradigm is the strongest signal — a repeating block means a programmatic
or conviction taker, not random flow.
- **If a Paradigm block tape is injected** into the session (via a block-trade context tool or
  equivalent feed): scan it for prior blocks matching this structure — same `strategy_code` +
  same leg geometry (underlying, expiry pattern, strike/width or moneyness) within 30d. Report:
  count of matching blocks, size range, most recent (date + level + side), and whether one-sided
  (single taker building) or two-way.
- **If no Paradigm tape is injected** (e.g. running outside the Dime terminal): say so in one
  line and fall back to identifying Paradigm-routed prints on the Deribit tape (see 3b). Never
  fabricate block counts.

### 3b — Deribit tape, always fetch (public, no auth)
Per leg:
`web_fetch GET /api/v2/public/get_last_trades_by_instrument?instrument_name=<leg>&count=1000&start_timestamp=<now_ms − 30d>&end_timestamp=<now_ms>&sorting=desc`
(fall back to `count=100&sorting=desc` if the windowed pull returns nothing).

**Identify Paradigm / block prints on the tape:** each trade carrying a `block_trade_id` field
is a block trade — Paradigm-routed flow surfaces here as blocks (and multi-leg blocks share one
`block_trade_id` with `block_trade_leg_count` > 1). Trades with no `block_trade_id` are on-screen.
Split them: block prints on the same leg/strike are the strongest cross-confirmation of the same
flow when the native Paradigm tape isn't injected.

Per leg report: total prints, of which blocks, total contracts, and most-recent timestamp (30d window).

### 3c — Flow impact (when the structure printed in multiple clips recently)
When a leg/structure has traded in several clips — especially same-day, same side — quantify the
accumulation footprint (this is what matters when one taker is working an order):
- **Clips:** number of fills and size of each (or total + size range).
- **Price impact:** how the leg's traded price and the underlying moved from first clip to latest
  (e.g. "6 clips, 20–50x, mark +14% and spot +1.1% as the taker lifted").
- **Vol & spread:** change in `mark_iv` and bid/ask width across the clips — is the taker paying up
  and widening the screen, or getting absorbed quietly? Pull the current ticker (`mark_iv`,
  `best_bid_price`/`best_ask_price`) and compare against the clip prices/times to read the impact.

Keep the *output* of this tight (one or two lines / a small table) — the depth is in the analysis,
not the word count.

### Secondary venues (optional)
Only when they add real signal — OKX (`/api/v5/market/trades`), Paradex (`paradex_trades` MCP for
perp legs). Do NOT pad the output with "not listed" rows for venues that never list the instrument.

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
3. **Prior Prints (30d)** — the headline (always fetched, per Step 3). Lead with a one-line
   verdict on recurrence: did this structure block on **Paradigm** before, and have the legs
   (incl. block prints) traded on **Deribit**?
   (e.g. "Seen 3× on Paradigm, last 29 May @ similar level; 6 block prints on Deribit, all today").
   If the structure printed in multiple clips recently, add a **Flow Impact** line/mini-table:
   clip count + sizes, and price / IV / spread drift since the taker started (per Step 3c).
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
