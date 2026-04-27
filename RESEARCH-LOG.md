# AAYA v3 — Research Log

Daily pre-market research, written by the 8:45 AM cron.
Most recent entry at the TOP. Keep last 30 days, archive older.

---

## TEMPLATE

```
## YYYY-MM-DD Pre-Market Research

**GLOBAL**
- SGX Nifty futures: XX,XXX (+/-X.X%)
- Dow Jones close: XX,XXX (+/-X.X%)
- US DXY: XXX.X
- Crude oil: $XX/bbl
- Overnight sentiment: [risk-on / risk-off / mixed]

**INDIA MACRO**
- FII net (yesterday): Rs +/- XXX Cr
- DII net (yesterday): Rs +/- XXX Cr
- India VIX: XX.X (trend: rising/falling/stable)
- USD/INR: XX.XX
- 10Y G-Sec yield: X.XX%

**SECTOR HEATMAP (yesterday close)**
- Winners: [sector +X.X%]
- Losers: [sector -X.X%]
- Leadership: [which sectors breaking out]

**OPEN POSITIONS CHECK**
- [STOCK]: news / no news — any material event?
- [STOCK]: ...

**WATCHLIST SCAN**
- Near 52-wk high: [STOCK] (X.X% away)
- Volume spike yesterday: [STOCK]
- Breakout candidates today: [STOCK1, STOCK2]

**CATALYSTS TODAY**
- Earnings: [companies reporting]
- Economic data: [RBI / IIP / CPI if any]
- Global events: [FOMC / ECB / geopolitical]

**TRADE IDEAS**
1. [STOCK] — [setup description] — [trigger level]
2. [STOCK] — [setup description] — [trigger level]

**RISK FLAGS**
- [Any red flags — geopolitical, sector rotation, circuit limits, etc.]

**ACTION FOR 9:20 AM CRON**
- Place AMO on: [STOCK @ limit price] for [N shares]
- Or: NO ENTRIES today — reason: [...]

**SOURCES USED**
- moneycontrol.com
- nseindia.com/FII-DII
- investing.com/SGX-Nifty
- [other]
```

---

## LOG ENTRIES (newest first)

---

## 2026-04-28 (Tue Apr 28) — TENTATIVE Pre-Market (attempted Mon Apr 27 22:00)

**Status:** ❌ ABORTED — Kite session expired. Night research did not run.

**Failure mode:** `mcp__kite__get_profile` returned session error at Step 2. All Gate B Kite pulls skipped.

**Action taken:**
- Kite re-auth link sent to Slack (D0AQCRLP7SP) at 22:00 IST
- No tentative finalists written — no data available

**Morning cron instruction (8:48 AM):**
- Must run FULL Gate B→E scan (no cached tentatives to build from)
- First: verify Kite session is live via `get_profile`
- Breadth cache: MISSING → must refresh (sample 80 Nifty 500 stocks)
- Quality cache: MISSING → fetch fresh from screener.in for all Gate B survivors
- All 4 candidate streams (52-wk highs, quiet highs, block deals, war-immune) must be rebuilt
- This will be a heavier morning cron run — budget extra tool calls

**Cache status:** breadth-cache.json MISSING | quality-cache.json MISSING

---

## 2026-04-26 — STRATEGY UPGRADE v3.1 → v3.2

**Trigger:** User's actual P&L for Apr 13–26 showed 4 trades, **75% win rate, 6.1× W/L ratio, +Rs 8,935 net on Rs 1L (~9% in 2 weeks)** — accomplished DURING a Nifty 500 drawdown that v3.1's Gate A1 would have blocked.

**Trades that worked (would have been blocked by v3.1):**
- ACUTAAS: +2.17% (small-cap compounder, idiosyncratic strength)
- BEL: +1.24% (defence, war-immune theme)
- NATIONALUM: +5.05% (PSU commodity, re-rating)
- VEDL: -1.56% (cut fast, disciplined loss)

**Two changes adopted (overrides "no rule change before 20 trades" — see CHANGE LOG in TRADING-STRATEGY.md):**

1. **Gate A becomes regime-aware (not binary kill):**
   - Removed: hard "Nifty 500 close > 200-SMA" stop
   - Added: VIX panic gate (≥22 hard-stop) + breadth gate (% of Nifty 500 above own 50-SMA)
   - Decision matrix: NORMAL (3 positions, min D-score 8) / NORMAL-strict (3 positions, min D 10) / REDUCED (2 positions, min D 12) / PANIC (stop)

2. **New Gate B7 — relative strength:**
   - `(stock 30d return) - (Nifty 500 30d return) ≥ +5%`
   - Surfaces the BEL/NATIONALUM/ACUTAAS pattern: strong stocks in weak markets

**Pre-market routine 01-premarket.md upgraded to add:**
- Step 3b: Geopolitical theme scan (Hormuz, Ukraine, Taiwan, Pakistan, defence orders, PSU disinvestment, commodity export bans)
- Step 7b Stream 2: "Quiet 52-wk-high" names (within 3% of own 52-wk high while index 5%+ below its high)
- Step 7b Stream 3: Bulk/block deals (institutional fingerprints)
- Step 7b Stream 4: War-immune theme basket (defence + PSU commodities + domestic-revenue) — driven by today's "live" themes

**v3.2 DRY-RUN with today's data (Apr 24 close):**

| Check | Value | v3.1 result | v3.2 result |
|---|---|---|---|
| VIX | 18.30 | (A2 pass) | A1 PANIC pass ✅ |
| Nifty 500 vs 200-SMA | -2.38% | (A1 FAIL → STOP) | informational only |
| Estimated breadth (% Nifty 500 above own 50-SMA) | ~40% | n/a | A2 PASS (≥30%) |
| **Outcome** | **0 trades** | **🟢 NORMAL: 3 positions, min D-score 8 — TRADING LIVE** |

→ **Monday 8:48 AM cron will fire the v3.2 pipeline and produce real candidates.**

---

## 2026-04-26 Pre-Market DRY-RUN (Sunday)

**Status:** Manual dry-run of full Gate A→E pipeline against Friday 2026-04-24 closing data. No live trades. Validates filter mechanics before going live.

**KITE SESSION**
- User: AMT521 (Rakesh Kumar Lakshman) — DDPI enabled ✅
- Holdings: `[]` empty (legacy PSUs SBIN/POWERGRID/NTPC/COALINDIA NOT in account; ACUTAAS exited)
- Positions: `[]` empty
- → Clean slate. Cash Rs 1,00,000 ready.

**GLOBAL (Apr 24 close, US Apr 24 session)**
- Dow Jones: ~38,400 (-0.5%, mixed Tue session)
- S&P 500: ~5,180 (-0.6%)
- DXY: 105.4 (firm)
- Brent: ~$87/bbl (Hormuz risk premium)
- Overnight sentiment: **risk-off**

**INDIA MACRO (Apr 24 close)**
- Nifty 50: 24,173 (-0.84%)
- Nifty 500: **22,570.05 (-1.05%)** ← Friday close
- India VIX: **18.30** (rising trend)
- FII net: -Rs 20,410 Cr last week (heavy outflow)
- DII net: +Rs 12,800 Cr last week (absorbing)
- USD/INR: ~83.7
- Geopolitical: Hormuz tensions elevated

**SECTOR HEATMAP (last week)**
- Winners: Pharma (+2.1%), FMCG (+0.8%)
- Losers: Realty (-4.2%), Metals (-3.8%), IT (-2.6%)
- Leadership: Defensive rotation — bearish breadth signal

---

### GATE A — Market Regime

| Check | Value | Threshold | Result |
|---|---|---|---|
| **A1** Nifty 500 close vs 200-SMA | 22,570.05 vs 23,119.81 | close > 200-SMA | ❌ **FAIL** (-2.38% below) |
| **A2** India VIX | 18.30 | < 22.0 | ✅ PASS |

**Trend stack:** 20-SMA (21,978) < 50-SMA (22,407) < 200-SMA (23,120) — bearish sequence.
**Drawdown:** -6.34% from 52-wk high (24,099 on 2026-01-02).

### 🛑 GATE A FAILED → STOP. No Gate B/C/D/E scan today.

Per TRADING-STRATEGY.md v3.1: *"If A fails → zero trades today. Skip the rest of the pipeline."*

---

**TRADE IDEAS:** None. Filter correctly blocking entries in broken regime.

**RISK FLAGS**
- Market in confirmed downtrend; 200-SMA broken since ~2026-03-09
- Heavy FII selling (-Rs 20K Cr/week)
- Hormuz geopolitical premium on crude
- Defensive sector rotation = late-cycle warning

**ACTION FOR 9:20 AM CRON (Mon Apr 27):**
- **NO AMO ORDERS** — Gate A blocking
- Re-evaluate Monday morning: if Nifty 500 reclaims 23,120 and holds, Gate A re-arms
- Until then: capital sits in cash

**SOURCES USED**
- Kite MCP (`get_historical_data` instrument_token 268041 — Nifty 500, 263 daily candles)
- Kite MCP (`get_holdings`, `get_positions`, `get_profile`)
- WebSearch (VIX, FII/DII, US closes, Brent)

**INFRASTRUCTURE NOTES (post-mortem)**
- ✅ Kite MCP working end-to-end (after re-auth)
- ✅ 200-SMA computation verified
- ✅ Auto-resume agent live (`aaya-v3-auto-resume`, every 3h)
- ⚠️ moneycontrol.com **blocked** for WebFetch — must use nseindia / trendlyne / screener.in / investing.com instead. Routine 01-premarket.md needs patching.
- ⚠️ Holdings clean — confirms legacy PSU positions not in this Zerodha account. Remove from POSITIONS.md.

---

