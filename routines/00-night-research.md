# 00 — Night Research (10:00 PM IST, Sun-Thu nights)

You are running the AAYA v3.4 NIGHT-RESEARCH cron. This fires at 22:00 IST on Sun, Mon, Tue, Wed, Thu — preparing tentative finalists for the NEXT trading morning.

## YOUR ROLE

Do the **heavy quantitative work** the night before, while end-of-day data is settled and the morning's usage window is preserved. Output a TENTATIVE candidate list to RESEARCH-LOG.md. The morning cron will refresh it with overnight context (US close, geopolitical, VIX) and finalise the green-list.

## WHY 22:00 IST

- NSE closed at 15:30 → 6.5h buffer for clean EOD data
- Still well before US session opens — no overnight gap noise yet
- Sun→Thu evenings cover Mon→Fri mornings respectively
- Friday night is excluded — no Saturday trade

## HARD RULES

- **DO NOT place any orders.** Execution is 02-market-open's job at 9:23 AM tomorrow.
- **DO NOT use Perplexity.** WebSearch + WebFetch + Kite MCP only.
- **moneycontrol.com is BLOCKED** for WebFetch. Use trendlyne / nseindia / investing.com / chartink / screener.in.
- If Kite session expired: call `mcp__kite__login`, post auth URL to Slack, exit. Morning cron will retry.
- **Cap the candidate pool at 25** before Gate B Kite pulls (cost discipline).
- **Cache slow-moving data** — breadth, 200-SMA, Gate C quality. Recompute weekly only.
- **Output TENTATIVE only.** Final decisions belong to morning cron after overnight refresh.

## STEP-BY-STEP

### Step 0 — Trading-day check

```
Read: /Users/rakesh/AAYA Calculation/aaya-v3/data/nse-holidays-2026.txt
Compute TOMORROW (next trading day after weekends/holidays).
If TOMORROW is a holiday → exit silently. No work to do.
If today is Friday/Saturday → exit silently (cron filter 0-4 should prevent this anyway).
```

### Step 1 — Read state

```
Read: /Users/rakesh/AAYA Calculation/aaya-v3/TRADING-STRATEGY.md
Read: /Users/rakesh/AAYA Calculation/aaya-v3/POSITIONS.md
Read: /Users/rakesh/AAYA Calculation/aaya-v3/RESEARCH-LOG.md (last 3 entries)
Read: /Users/rakesh/AAYA Calculation/aaya-v3/PERFORMANCE.md (headline stats)
```

### Step 2 — Kite session

```
Call: mcp__kite__get_profile
If error → Slack to D0AQCRLP7SP with re-auth URL → exit. Morning cron retries.
```

### Step 3 — Cache check (slow-moving data)

```
Read: /Users/rakesh/AAYA Calculation/aaya-v3/data/breadth-cache.json (if exists)
If cache > 6 days old OR missing → REFRESH it (Step 3a)
Else → use cached values, skip 3a

Read: /Users/rakesh/AAYA Calculation/aaya-v3/data/quality-cache.json (if exists)
If cache > 6 days old OR missing → REFRESH on Sundays only (Step 3c)
Else → use cached values
```

### Step 3a — Breadth + 200-SMA refresh (weekly, Sundays only)

Only run if today is Sunday OR cache is stale.

```python
# Sample 80 random Nifty 500 constituents (statistically tighter than 50)
# For each:
#   mcp__kite__get_historical_data (daily, 60 candles)
#   compute sma_50, check close > sma_50
breadth_pct = above_50sma / 80

# Also recompute Nifty 500 200-SMA from instrument 268041
sma_200 = mean(last 200 closes)

# Cache to data/breadth-cache.json:
{
  "computed_at": "2026-04-26T22:05:00+05:30",
  "expires_at": "2026-05-03T22:05:00+05:30",  # 7 days
  "nifty500_breadth_pct": 0.40,
  "nifty500_sma_200": 23119.81
}
```

### Step 3b — Recent volume + price (always — 1 daily candle per stock)

For each Nifty 500 stock (sampled or top 50 by recent activity), pull just yesterday's daily candle:
- mcp__kite__get_quotes (batch — up to 500 symbols per call)
- Gives us close, volume, day high/low

This is enough for Step 4 pre-filtering without 260-candle pulls.

### Step 4 — Build candidate pool (cap at 25)

Apply 4 streams as in v3.3, but **prune to 25 BEFORE Gate B Kite pulls**.

**Stream 1 — 52-wk highs today:** WebFetch nseindia / trendlyne
**Stream 2 — Quiet 52-wk-high names:** stocks within 3% of own 52-wk high while index 5%+ below its high (cheap — uses already-fetched LTP + recent highs from get_quotes)
**Stream 3 — Bulk/block deals last 5 sessions:** WebFetch nseindia bulk-deals
**Stream 4 — War-immune basket (today's geopolitical themes):** WebSearch (1 query for theme detection, then map to canonical war-immune list)

**Union all 4 streams, dedupe, intersect with Nifty 500 list.**

**Pre-Gate-B prune to TOP 25 by relative-strength proxy:**
```python
# 5-day return = (close_today - close_5d_ago) / close_5d_ago
# Computable from already-fetched 60-candle data, ~free
rs_proxy = stock_5d_return - nifty500_5d_return
candidates = top_25_by(rs_proxy)
```

### Step 5 — Gate B (Momentum, 7 conditions, ALL must pass)

For each of the 25 candidates, pull 260 daily candles via `mcp__kite__get_historical_data`. Compute B1–B7 per v3.3 spec:

```
B1: within 5% of 52-wk high
B2: ≥3 of last 5 sessions had volume ≥1.5× 20-day avg
B3: 50 ≤ RSI(14) ≤ 80
B4: close > SMA(50) AND SMA(50) rising
B5: close > SMA(200)
B6: daily traded value ≥ Rs 50 lakh
B7: stock 30d return - Nifty 500 30d return ≥ +5%
```

Drop failures. Expected survivors: 5–12 stocks.

### Step 6 — Gate C (Quality)

```
On Sundays:
  For each survivor: WebFetch screener.in/company/[SYMBOL]/
  Compute C1–C6, cache to data/quality-cache.json with 7-day expiry.

On Mon-Thu nights:
  Read cached quality data per stock from data/quality-cache.json.
  If a survivor isn't in cache → fetch fresh and append (rare case for new candidate).
```

Drop failures. Expected finalists: 3–7 stocks.

### Step 7 — Gate D (Scoring) — consolidated single-query per stock

For each finalist, run **ONE consolidated WebSearch** instead of 4 separate queries:
```
WebSearch: "[STOCK] market share India FII holding promoter buying 2026 sector ranking"
```
Parse for D1/D2 (monopoly/duopoly), D5 (sector hot), D8 (promoter buying), D9 (FII up).

Compute D3 (high quality metric) and D4 (revenue YoY ≥20%) from cached quality data (Gate C).
Compute D6 (fresh 52-wk high today), D7 (vol yesterday ≥2.5× avg) from already-fetched Kite data — free.

Total: 1 WebSearch per finalist instead of 4.

Rank by D-score DESC.

### Step 8 — Apply Gate A regime + Gate E preview

Gate A regime mode determines max_positions and min_d_score, but **Gate A2 (breadth) is from cache** — morning cron re-validates with live VIX.

Apply E1 (max 2 same-sector), E3 (skip held), E4 (skip too-expensive). Truncate to top 6 (overshooting max_positions=3 to give morning cron flexibility for invalidations).

### Step 9 — Compute entry levels for each tentative finalist

```python
entry_target = ltp * 1.001   # tick-rounded to Rs 0.05
position_size = floor(33000 / entry_price)
atr_14 = compute from 260 candles
stop = entry - 1.5 * atr_14   # floor 3%, ceiling 6%
target_t1 = entry * 1.04
target_t2 = entry * 1.07
```

### Step 10 — Write TENTATIVE entry to RESEARCH-LOG.md

Prepend today's entry with:

```markdown
## YYYY-MM-DD (Mon Apr 28) — TENTATIVE Pre-Market (computed Sun Apr 27 22:05)

**Status:** TENTATIVE. Morning cron at 8:48 AM will refresh with overnight US close + geopolitical + live VIX, then invalidate or finalise.

**REGIME (cached, will re-check live in morning):**
- Cached breadth: XX% above own 50-SMA (refreshed YYYY-MM-DD)
- Cached Nifty 500 200-SMA: XX,XXX
- Tentative mode: NORMAL / NORMAL-strict / REDUCED

**TENTATIVE FINALISTS (top 6 by D-score, max_positions=3 will be selected morning):**

| Rank | Symbol | D-score | Entry | Stop | T1 | T2 | Qty | Sector | RS (30d) | Notes |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | ACUTAAS | 14 | 2,395 | 2,302 | 2,491 | 2,563 | 13 | Pharma sm-cap | +12% | ... |
| 2 | ... | | | | | | | | | |

**INVALIDATION CRITERIA (morning cron applies these):**
- Overnight US session: Dow closes < -2% → consider PANIC mode
- VIX live above 22 → PANIC, drop all
- Stock-specific overnight news: SEBI/promoter exit/auditor → drop that stock
- Gap-up >2% on any finalist → skip that one
- Gap-down >1% → skip that one
```

### Step 11 — Slack notification (informational)

```
🌙 AAYA v3.4 Night-Research [date]
Tentative finalists for tomorrow's open: N
Top: [STOCK1] (D=14), [STOCK2] (D=12), [STOCK3] (D=10)
Mode: 🟢 NORMAL
Morning cron at 8:48 AM will refresh + finalise.
```

Or if 0 finalists: silent (don't spam).

### Step 12 — Commit and push

```
cd "/Users/rakesh/AAYA Calculation/aaya-v3"
git add -A && git commit -m "night research YYYY-MM-DD" && git push origin main
```

## OUTPUT CONTRACT

- Cache hits / misses: [breadth: hit/miss, quality: hit/miss]
- Candidates evaluated: N
- Survivors past Gate B: N
- Survivors past Gate C: N
- Tentative finalists ranked: N
- Slack sent: yes/no
- Git pushed: yes/no
- Tool calls used (rough): N

## FAILURE MODES

| Failure | Response |
|---|---|
| Kite session expired | Slack expiry note, exit. Morning cron retries. |
| Cache file corrupted | Treat as miss, refresh. Log warning. |
| 0 candidates pass Gate B | Write "no setup tonight" to RESEARCH-LOG.md, exit. Morning cron will see this and skip. |
| Usage window pressure (>50% used) | Stop after Gate B, write partial finalists, log warning. Morning cron promotes them with extra caution. |
| WebFetch sources all blocked | Log, fall back to WebSearch, continue. |
