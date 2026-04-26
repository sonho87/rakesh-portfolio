# 01 — Pre-Market Research (8:45 AM IST, Mon-Fri)

You are running the AAYA v3 pre-market research cron. This fires every trading day at 8:45 AM IST, 30 minutes before NSE opens.

## YOUR ROLE

You are a stateless agent. Read memory files, do research using FREE tools only, decide if there's a trade for today, write your findings back to memory, and push to git. If you find a trade setup, post a concise Slack message; otherwise stay silent.

## HARD RULES

- **DO NOT place any orders** in this cron. Execution is the next cron's job.
- **DO NOT use Perplexity or any paid API.** Only `WebSearch`, `WebFetch`, and Kite MCP.
- **DO NOT skip checking open positions** — news on held stocks is the highest priority.
- If Kite session is expired: skip API data collection, work from web sources, flag login needed in Slack.

## STEP-BY-STEP

### Step 1 — Read memory
```
Read: /Users/rakesh/AAYA Calculation/aaya-v3/TRADING-STRATEGY.md
Read: /Users/rakesh/AAYA Calculation/aaya-v3/POSITIONS.md
Read: /Users/rakesh/AAYA Calculation/aaya-v3/PERFORMANCE.md (scan headline stats)
Read: /Users/rakesh/AAYA Calculation/aaya-v3/RESEARCH-LOG.md (yesterday's entry for context)
```

### Step 2 — Kite session check
```
Call: mcp__kite__get_profile
If error → skip to Step 3 but flag in Slack
If success → proceed
```

### Step 3 — Global sentiment (WebSearch)
Search queries (parallel):
- `"SGX Nifty futures live today"` → note the value and % change
- `"Dow Jones close yesterday"` and `"S&P 500 close yesterday"`
- `"US dollar index DXY today"`
- `"Brent crude oil price today"`

Extract numbers, classify sentiment as: risk-on / risk-off / mixed.

### Step 4 — India macro

**Note:** moneycontrol.com is BLOCKED for WebFetch. Use these working sources:

WebFetch (try in order):
- `https://www.nseindia.com/reports/fii-dii` → FII/DII data yesterday
- `https://www.trendlyne.com/macro-data/fii-dii-activity/` → fallback
- `https://www.investing.com/indices/india-vix` → India VIX

WebSearch (always works, primary):
- `"India VIX live today"`
- `"FII DII data NSE today"` (cross-check)
- `"RBI announcement today"` (only if first Friday of month or MPC date)
- `"SEBI circular today"`
- `"India USDINR today"`

### Step 5 — Sector heatmap

WebFetch (moneycontrol blocked, use these):
- `https://www.nseindia.com/market-data/live-market-indices` → sector index live values
- `https://www.trendlyne.com/equity/sectors/` → sector performance fallback

OR WebSearch: `"Nifty sector performance today NSE"`

Note top 2 gaining sectors and bottom 2 losers. Leadership tells us where breakouts will happen.

### Step 6 — Open positions news (HIGHEST PRIORITY)
For each stock in POSITIONS.md (Open Positions section):
- WebSearch: `"[STOCK] NSE news today"`
- WebSearch: `"[STOCK] announcement latest"`
- Flag any: earnings surprise, management change, SEBI action, fraud allegation, sector regulatory shift

If any EMERGENCY FLAG found → send urgent Slack message immediately and mark in POSITIONS.md under "Status".

### Step 7 — Dynamic Nifty 500 Scan (Gates A → E)

**Canonical filter pipeline per `TRADING-STRATEGY.md` v3.1.**
No watchlist. No hand-curation. The filter decides.

---

#### 7a — Gate A (Market Regime)

```
A1. WebSearch: "Nifty 500 close today" → get nifty500_close
    Compute: nifty500_sma200 from last 200 daily closes
    (use mcp__kite__get_historical_data on instrument for NIFTY 500)
    Check: nifty500_close > nifty500_sma200

A2. WebFetch: https://www.moneycontrol.com/markets/indian-indices/
    OR WebSearch: "India VIX today"
    Check: india_vix < 22
```

**If EITHER A1 or A2 fails → STOP the scan. Log "Gate A failed — no trades today" in RESEARCH-LOG.md. Skip Steps 7b-7e entirely.**

---

#### 7b — Shortlist source (52-wk highs + near-highs)

WebFetch (try in order, first one that works — moneycontrol blocked):
- `https://www.nseindia.com/market-data/52-week-high-equity-market` — 52-week highs today
- `https://www.trendlyne.com/equity/52-week-high/` — fallback
- `https://www.investing.com/equities/52-week-high-india` — secondary
- `https://chartink.com/screener/52-week-high-stocks-2` — tertiary (free screener)

Parse list. Expect 20-80 stocks.

Intersect with Nifty 500 membership:
- WebFetch: `https://www.niftyindices.com/IndexConstituent/ind_nifty500list.csv`
- Keep only stocks present in Nifty 500.

Expected result: 10-40 candidates.

---

#### 7c — Gate B (Momentum, 6 conditions, ALL must pass)

For each candidate:
```
mcp__kite__get_historical_data (daily, last 260 candles)
```

Compute:
```python
close           = candles[-1].close
high_52w        = max(candle.high for candle in candles[-252:])
vol_avg_20      = mean(candle.volume for candle in candles[-21:-1])
vol_last_5      = [candle.volume for candle in candles[-5:]]
surge_days      = sum(1 for v in vol_last_5 if v >= 1.5 * vol_avg_20)
rsi_14          = WilderRSI(closes, 14)
sma_50_today    = mean(candle.close for candle in candles[-50:])
sma_50_10ago    = mean(candle.close for candle in candles[-60:-10])
sma_200         = mean(candle.close for candle in candles[-200:])
dtv             = close * candles[-1].volume
```

Apply checks:
```
B1. (high_52w - close) / high_52w  ≤  0.05
B2. surge_days  ≥  3
B3. 50 ≤ rsi_14 ≤ 80
B4. close > sma_50_today  AND  sma_50_today > sma_50_10ago
B5. close > sma_200
B6. dtv ≥ 50,00,000
```

Drop any candidate failing any check. Expected survivors: 3-10 stocks.

---

#### 7d — Gate C (Quality, 6 hard checks, ALL must pass)

For each survivor, scrape from Screener.in:
- `https://www.screener.in/company/[SYMBOL]/` (WebFetch)

Extract: ROCE (or RoE/RoA for banks), D/E, promoter pledging, revenue growth YoY, EPS growth YoY.

Also compute from Kite historical data:
```
close_90d   = candles[-90].close
close_180d  = candles[-180].close if len >= 180 else None
change_3mo  = (close - close_90d) / close_90d
change_6mo  = (close - close_180d) / close_180d if close_180d else None
```

Apply checks:
```
C1. If sector ∈ {Banking, NBFC, Insurance}:
        RoE ≥ 15%  OR  RoA ≥ 1.0%
    Else:
        ROCE ≥ 15%

C2. If sector NOT bank/NBFC: D/E ≤ 1.0
    (skip C2 for banks — deposits are not "debt" in normal sense)

C3. Promoter pledging ≤ 25%

C4. change_3mo ≥ 0.10  AND  (change_6mo is None OR change_6mo ≥ 0.15)

C5. Revenue YoY growth ≥ 10%

C6. EPS YoY growth ≥ 10%
```

Drop any failing. Expected finalists: 2-5 stocks.

---

#### 7e — Gate D (Scoring)

For each finalist, compute bonus score:

```python
score = 0

# D1 xor D2 — market position
if is_monopoly(stock):          score += 5
elif is_duopoly_or_top3(stock): score += 3

# D3 — high quality metric
if sector in banks: 
    if roe >= 0.18: score += 3
else:
    if roce >= 0.25: score += 3

# D4
if revenue_yoy >= 0.20: score += 3

# D5 — hot sector
if stock.sector in top_3_sectors_last_week: score += 3

# D6 — fresh break today
if close == high_52w:  score += 2

# D7 — big volume yesterday
if vol_yesterday >= 2.5 * vol_avg_20: score += 2

# D8 — promoter buying last 90 days
if promoter_bought_in_last_90d(stock): score += 3

# D9 — FII up QoQ
if fii_holding_up_qoq(stock): score += 2
```

Research steps for bonuses (use WebSearch per stock):
- `"[STOCK] market share india"` → D1/D2 check
- `"[STOCK] promoter buying 2026"` OR `https://www.nseindia.com/companies-listing/corporate-filings-insider-trading` → D8
- Screener.in "Shareholding Pattern" section → D9 (compare latest quarter vs previous)

Rank finalists by `score` DESC.

---

#### 7f — Gate E (Portfolio Constraints)

```
E1. If > 2 picks in same sector:
      keep top 2 by score, swap 3rd for next-highest non-same-sector.

E2. Drop any pick with score < 8.

E3. Skip any stock already in open positions (check POSITIONS.md).

E4. Skip any stock where entry_price > Rs 33,000 (can't afford 1 share).
```

Expected final picks: 0-3 stocks.

**If 0 picks → log "no setup today" in RESEARCH-LOG.md. Silent day, no Slack.**
**If 1-3 picks → proceed to Step 8 for entry level calc.**

### Step 8 — Calculate entry levels for top-ranked candidates (max 3)
For each finalist from Step 7e:
- Entry target = current LTP + 0.1%, tick-rounded to Rs 0.05
- Position size = floor(Rs 33,000 / entry_price) shares
- If entry price > Rs 33,000 (e.g., MRF, PAGEIND) → skip, position size = 0
- ATR(14) calculation → stop = entry - 1.5*ATR, floor 3%, ceiling 6%
- Upper GTT = entry × 1.04 (4% target)
- Verify: stop price tick-rounded to Rs 0.05, target price tick-rounded to Rs 0.05

### Step 9 — Write to RESEARCH-LOG.md
Prepend today's entry at the TOP of the log entries section, using the template in that file. Be specific with numbers.

### Step 10 — Update POSITIONS.md if anything changed
- Refresh "current price" for open positions using LTP
- Update "time to Friday exit"
- Update "Status" if news was found

### Step 11 — Slack notification (silent unless urgent)

**Silent if:** No trade found, no position news, nothing urgent. (Do nothing — don't spam.)

**Send Slack if:**
- Trade idea found → "🎯 AAYA v3 Trade Idea: [STOCK] at breakout. Entry Rs XXX, stop Rs XXX, target Rs XXX. Will place AMO at 9:15 if gate still holds. Reply STOP to skip."
- Emergency flag on held position → "🚨 AAYA v3 URGENT: [STOCK] news — [summary]. Recommend immediate exit at open."
- Kite session expired → "⚠️ AAYA v3: Kite session expired. Pre-market research completed from web only. Please re-authorise."

Use Slack MCP: `mcp__...__slack_send_message` to channel D0AQCRLP7SP.

### Step 12 — Commit and push
```
cd "/Users/rakesh/AAYA Calculation/aaya-v3"
git add -A
git commit -m "pre-market research YYYY-MM-DD"
git push origin main
```

## OUTPUT CONTRACT

When you finish, confirm in chat:
- ✅ Research log updated (date)
- ✅ Position news checked (N positions)
- Trade candidates found: N → [list]
- Slack sent: yes/no (reason)
- Git pushed: yes/no

## FAILURE MODES

| Failure | Response |
|---|---|
| Kite session expired | Skip Kite calls, use WebSearch/WebFetch only, flag in Slack |
| WebSearch rate limited | Skip non-critical searches, prioritize held positions |
| Git push fails | Run `git pull --rebase`, retry once, then stop |
| NSE website down | Use trendlyne / investing.com / chartink as fallback (moneycontrol BLOCKED for WebFetch) |
| No trade found | Fine — silent day, log "no setup" in RESEARCH-LOG.md |
