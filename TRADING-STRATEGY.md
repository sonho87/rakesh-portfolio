# AAYA v3.4 — Trading Strategy (The Rulebook)

**Last updated:** 2026-04-27
**Version:** 3.4 (research split: night-cron heavy + morning-cron light)
**Status:** ACTIVE
**Capital:** Rs 1,00,000
**Hold period:** Maximum 1 week (Mon entry → Fri exit mandatory)

---

## CORE PHILOSOPHY

- Follow NSE sector momentum, buy breakouts not dips
- Use ATR-based stops, never fixed % stops
- Never hold over the weekend (geopolitical gap risk)
- Trade less than you think you should (patience > activity)
- 3 concentrated positions, never 10+ diluted
- Filter decides the stock. We do not maintain a hardcoded watchlist.

---

## CAPITAL ALLOCATION

| Setting | Value |
|---|---|
| Total capital | Rs 1,00,000 |
| Max positions simultaneously | 3 |
| Capital per trade | Rs 33,000 (~33%) |
| Max per single stock | Rs 40,000 (40% — only for highest conviction) |
| Minimum per trade | Rs 25,000 (below this = not worth friction) |
| Cash buffer | Rs 5,000 (for GTT slippage / partial fills) |

---

## STOCK UNIVERSE

**Universe:** Nifty 500 (full breadth — PSUs, commodities, mid/small-caps all eligible)

No hardcoded watchlist. Every morning the pre-market cron scans the full Nifty 500 through Gates A→E. Whatever passes, we trade. If nothing passes, we don't trade that day.

**Examples of stocks that have historically qualified** (illustrative, not a watchlist):
NATIONALUM, VEDL, MOTHERSON, MAHABANK, Accutas — showing the filter works across PSUs, commodities, auto ancillaries, banks, and small-cap compounders.

**Philosophy:** "Monopoly/duopoly with quality" is a PROPERTY to verify on candidates, not a pre-curated list. A PSU can qualify. A Nifty 50 name can be skipped. The filter is the decision-maker.

---

## ENTRY PIPELINE — Gates A → E + Deep Research red-flag stage (v3.3)

After Gate E produces 0–3 finalists, a NEW stage runs **between pre-market (8:48 AM) and market-open (9:23 AM):**

### Stage F — Deep Research Red-Flag Scan (NEW v3.3, runs at 9:00 AM)

A dedicated cron (`aaya-01b-deep-research`) per-stock investigates each finalist for hard red flags. ANY red flag → drop the stock, no entry.

```
F1. NEWS SCAN (last 30 days)
    WebSearch: "[STOCK] news"
    HARD RED FLAGS (drop):
      - SEBI / RBI / income-tax investigation announced
      - Promoter resignation / arrest
      - Auditor resignation
      - Material loss / accident / fire / litigation
      - Major customer loss (>20% of revenue)
      - Credit rating downgrade

F2. CATALYST CALENDAR
    HARD RED FLAGS (drop):
      - Earnings call within next 5 trading days (gap risk)
      - Ex-dividend within next 3 days (price drop priced in)

F3. INSIDER ACTIVITY (NSE corporate filings)
    HARD RED FLAGS (drop):
      - Promoter selling > 0.5% of float in last 30 days
    SOFT SIGNALS (note, don't drop):
      - Promoter buying = +1 conviction note in RESEARCH-LOG

F4. RETAIL MANIA CHECK
    WebSearch "[STOCK] stock twitter today"
    HARD RED FLAG (drop):
      - Parabolic retail mania detected (top trending, multiple viral threads)
    Reasoning: retail-driven parabolas are top signals, not entry signals.

F5. ANALYST SANITY CHECK
    WebSearch "[STOCK] brokerage target price 2026"
    SOFT (don't drop, just note): at least one upgrade in last 30 days

F6. SECTOR ROTATION CHECK
    Cross-reference morning sector heatmap.
    HARD RED FLAG (drop):
      - Sector was top-3 performer 90 days ago AND is now bottom-3
        (clear rotation OUT of the theme)
```

**Output:** Either GREEN (passes all hard red flags → 02-market-open places the order) or RED (dropped → reason logged in RESEARCH-LOG.md, Slack note).

The deep-research agent is allowed to be slightly imperfect — better to occasionally drop a valid setup than to enter a stock with a known red flag.

---

## ENTRY PIPELINE — Gates A → E (ALL must pass)

### Gate A — Market Regime (daily check, regime-aware not binary)

**v3.2 change:** Replaced the old hard "Nifty 500 > 200-SMA" stop. That rule killed the system when relative-strength alpha is highest (weak index, strong stocks). Replaced with a panic gate + breadth gate that scales position count instead of nuking it.

```
A1. India VIX < 22                                 # PANIC GATE — hard stop
A2. Breadth: ≥ 30% of Nifty 500 stocks above       # MARKET INTERNALS
    their own 50-SMA
```

**Decision matrix:**

| A1 (VIX) | A2 (Breadth) | Mode | Max positions | Min D-score |
|---|---|---|---|---|
| FAIL (≥22) | any | 🛑 PANIC — STOP | 0 | n/a |
| PASS | PASS (≥30%) | 🟢 NORMAL | 3 | 8 |
| PASS | FAIL (<30%) AND VIX > 18 | 🟡 REDUCED | 2 | 12 (only top setups) |
| PASS | FAIL (<30%) AND VIX ≤ 18 | 🟢 NORMAL | 3 | 10 (slightly stricter) |

The breadth check (% of Nifty 500 above own 50-SMA) is computed by sampling the constituents via Kite historical data. In REDUCED mode we keep trading but require higher conviction — exactly the regime where defence/PSU/war-immune names tend to outperform.

---

### Gate B — Stock Momentum (7 conditions, all must pass)
```
B1. (high_52w - close) / high_52w ≤ 0.05                        # within 5% of 52-wk high
B2. count(sessions where volume ≥ 1.5 × vol_avg_20,
          over last 5 sessions) ≥ 3                              # sustained volume, not one-off
B3. 50 ≤ RSI(14) ≤ 80                                            # momentum zone, wider to allow trending
B4. close > SMA(50)  AND  SMA(50) > SMA(50) ten sessions ago     # 50-SMA rising
B5. close > SMA(200)                                             # stock in long-term uptrend
B6. daily_traded_value ≥ Rs 50,00,000                            # liquidity floor
B7. (stock 30-day return) - (Nifty 500 30-day return) ≥ +5%      # RELATIVE STRENGTH (NEW v3.2)
```

**Why B7 matters (v3.2 addition):** This is the single most important new check. It surfaces names that are outperforming the index — exactly the BEL/NATIONALUM/ACUTAAS pattern that worked Apr 13-26 when index was -1% but those names were +3-5%. Stocks with idiosyncratic strength (defence themes, commodity cycles, small-cap compounders) will pass even when the broader market is weak. Stocks merely riding a rising tide will fail.

---

### Gate C — Quality (6 hard checks, ALL must pass — any fail = reject)
```
C1. If sector ∈ {Banking, NBFC, Insurance}:
        RoE ≥ 15%  OR  RoA ≥ 1.0%                                # banking exception
    Else:
        ROCE (last FY) ≥ 15%

C2. Debt-to-Equity ≤ 1.0                                         # skip for banks (deposits ≠ debt)

C3. Promoter pledging ≤ 25%

C4. 3-month price change ≥ 10%   AND                             # strong recent leader
    6-month price change ≥ 15%                                   # sustained, not one-spike

C5. Revenue YoY growth (last FY) ≥ 10%                           # earnings back price

C6. EPS YoY growth (last FY) ≥ 10%                               # profitability growing
```

---

### Gate D — Scoring (bonus points, used for ranking)

Apply ALL applicable bonuses. Pick top 3 by total score (after Gate E).

| # | Criterion | Points |
|---|---|---|
| D1 | Monopoly (dominant market share in primary segment) | +5 |
| D2 | Duopoly / top-3 in segment (D1 xor D2) | +3 |
| D3 | ROCE ≥ 25% (or RoE ≥ 18% for banks) | +3 |
| D4 | Revenue YoY growth ≥ 20% | +3 |
| D5 | Sector in top-3 performing sectors last week | +3 |
| D6 | 52-wk high reached today (fresh break, not retest) | +2 |
| D7 | Volume yesterday ≥ 2.5 × vol_avg_20 | +2 |
| D8 | Promoter buying in last 90 days (NSE insider disclosures) | +3 |
| D9 | FII holding increased QoQ (latest shareholding pattern) | +2 |

**Max score:** 23  
**Minimum score to enter:** 8 (below this = weak setup, skip)

---

### Gate E — Portfolio Constraints (applied to Gate D ranking)
```
E1. Max 2 stocks from same sector in final 3 picks.
    → If >2 in a sector: keep top 2 by D-score, swap 3rd for next-highest non-same-sector.

E2. Max 1 stock with D-score < 10 per entry batch.
    → Prevents loading up on weak setups.

E3. Skip any stock already in open positions (no doubling).

E4. Skip any stock that cost > Rs 33,000 per share (unaffordable).
```

---

### ORDER TYPE — Hybrid entry (v3.3)

We are a momentum/breakout strategy. We buy AT or slightly ABOVE current price to confirm the trend. We never "wait for the dip" because that's how falling knives are caught. Falling-knife risk is already mitigated by Gates B1, B3, B4, B5, B7, C4 — a stock that passes those checks cannot mathematically be a knife.

But we can still get smarter on entry timing. v3.3 introduces a two-stage entry:

```
PRIMARY ENTRY (placed by 02-market-open, ~9:23 AM):
  Regular LIMIT BUY @ LTP + 0.1%, tick-rounded to Rs 0.05, validity DAY
  variety: regular   (NOT amo — market is already open at 9:23 AM,
                      AMO orders are rejected during market hours.
                      AMO is only valid 4:00 PM – 8:55 AM next day.)
  → tries to fill near opening auction price
  → most fills happen here on a healthy open

FALLBACK ENTRY (placed only if primary unfilled by 9:30 AM):
  HARD GUARD: If today is FRIDAY → skip the fallback entirely.
              A Friday-only fallback would force an exit at the 04-eod cron
              (3:11 PM same day), guaranteeing a ~0.3% round-trip cost loss
              with no time for the thesis to play out. Primary fill on Friday
              or no entry.
  Conditions to qualify (Mon-Thu only):
    (a) Stock NOT down >1% from prev close (knife guard)
    (b) Stock NOT up >2% from prev close (chase guard)
    (c) Stock STILL passes Gate B7 (relative strength) at this LTP
    (d) VIX has not spiked above 22 since pre-market check
  If all 4 met:
    Fresh regular LIMIT BUY @ current LTP + 0.05%, tick-rounded, validity DAY
    (one fallback attempt only; no further retries that day)
  Else:
    Cancel primary, log "skipped — entry conditions failed", no entry today.

HARD SKIPS (any time during entry window):
  - Opening gap > 2% above prev close → skip (chase rule)
  - Stock down > 1% from prev close → skip (knife guard NEW v3.3)
  - VIX jumps to ≥ 22 → skip + cancel any pending order (regime flip)
  - NSE trading holiday → all crons exit silently (see data/nse-holidays-2026.txt)
```

This still avoids falling knives (down >1% = skip), but lets us pick up healthy intraday pullbacks (down 0–1%) that would otherwise miss the opening fill window. Never market orders at entry except for the same-day re-entry within 30 min of a rejection.

---

## EXIT RULES

### Automated via GTT OCO (placed immediately after fill)
| Leg | Level | Action |
|---|---|---|
| Upper (take profit, tier 1) | Entry × 1.04 | Sell 50% of position |
| Upper (take profit, tier 2) | Entry × 1.07 | Sell remaining 50% |
| Lower (stop loss) | Entry − 1.5 × ATR(14) — floor 3%, ceiling 6% | Full exit |

### Semi-automated via Slack ping
| Trigger | Action |
|---|---|
| Position up ≥ 2% at midday (1 PM scan) | Bot pings Rakesh → Rakesh moves GTT stop to break-even |
| Tier 1 target hit (+4%) | Bot confirms 50% sell, pings Rakesh to move stop to entry + 1% |

### Mandatory time stop
- **Friday 3:10 PM** — cancel all open GTTs → place MARKET SELL for everything
- Never, under any circumstances, hold a position into the weekend

### Emergency exits (full position, market order, any time)
- SEBI investigation initiated
- Promoter fraud / accounting restatement
- War / major geopolitical event (Middle East escalation, India-Pakistan, Taiwan)
- Stock-specific catastrophic news (CEO exit, license cancellation)

---

## SIZING FORMULA

```
shares = floor(Rs 33,000 / entry_price)
actual_capital = shares × entry_price
```

If entry_price > Rs 33,000 → skip (can't afford minimum whole share).

---

## PROHIBITED ACTIONS (hard rules — never override)

- ❌ Never use options or futures
- ❌ Never hold over the weekend
- ❌ Never enter a position without a GTT stop placed
- ❌ Never add to a losing position (no averaging down)
- ❌ Never cancel a stop loss on a "feeling"
- ❌ Never chase an entry — if the opening gap is >2%, skip the trade
- ❌ Never override the filter — if a stock doesn't pass A→E, it doesn't matter who recommended it
- ❌ Never trade a stock with D-score < 8

---

## WIN RATE GATES

| Win Rate (after 20+ trades) | Outcome | Action |
|---|---|---|
| < 43% | Losing money after tax/costs | 🚨 STOP trading, review thesis |
| 43-50% | Breaking even | ⚠️ Continue but reduce size 50% |
| 50-55% | Matching Nifty roughly | 🟡 Continue, marginal edge |
| 55-60% | Beating Nifty 14-15% | 🟢 Healthy, scale gradually |
| > 60% | Strong edge | 🚀 Consider scaling capital to Rs 2-3L |

---

## CHANGE LOG

| Date | Version | Change | Reason |
|---|---|---|---|
| 2026-04-24 | v3.0 | Initial — 1-week rotation, ATR stops, GTT OCO, Friday time stop | Moved from v2 buy-and-hold to active rotation per user directive |
| 2026-04-24 | v3.1 | Filter tune: B1 tightened to 5%, RSI cap 80, volume 3-of-5 days, 200-SMA per stock, banking exception, 3/6-month combo, Revenue+EPS YoY gates, promoter buying +FII bonuses, sector diversification (Gate E) | User evaluation of v3.0 filter — six corrections adopted, one rejected (analyst upgrades — lagging indicator) |
| 2026-04-26 | v3.2 | (1) Replaced hard "Nifty 500 > 200-SMA" Gate A1 with regime-aware decision matrix (VIX panic gate + breadth-based position scaling). (2) Added Gate B7 — relative strength (stock 30d return − index 30d return ≥ +5%). | **Evidence override of "no rule change before 20 trades"**: User's manual P&L Apr 13–26 (4 trades, 75% win rate, 6.1× W/L ratio, +Rs 8,935 net on Rs 1L = ~9% in 2 weeks) demonstrably worked DURING a Nifty 500 drawdown. Stocks traded (BEL, NATIONALUM, ACUTAAS, VEDL) all had idiosyncratic strength independent of index. Old Gate A1 would have blocked all 4 entries. Removing a rule that demonstrably blocks profitable setups is conservative, not aggressive. |
| 2026-04-26 | v3.3 | (1) Hybrid entry: AMO LIMIT primary + intraday fallback LIMIT if primary unfilled by 9:30 AM, with explicit knife-guard (skip if stock down >1%) and chase-guard (skip if up >2%). (2) New Stage F — Deep Research red-flag scan (cron `aaya-01b-deep-research` at 9:00 AM) inspects each finalist for SEBI/RBI actions, promoter exits, auditor resignations, earnings within 5 days, promoter selling >0.5% float, retail mania, sector rotation. ANY hard red flag → drop. | User raised valid concerns about (a) entry mechanics ("are we waiting for dip or buying market?") and (b) need for a per-stock research agent that "may not be perfect but we will reach somewhere." Hybrid entry catches healthy pullbacks without catching knives; deep-research stage prevents the obvious "I should have known" losses (SEBI announcements, ex-promoter dumps, earnings gaps). Both upgrades layered ON TOP of v3.2 filter — no existing rule weakened. |
| 2026-04-27 | v3.4 | **Architecture split — research moved to night cron.** (1) NEW cron `aaya-00-night-research` at 22:00 IST Sun-Thu does ~70% of the heavy work: Gates A2 (breadth), B (momentum), C (quality, cached weekly), D (scoring). Outputs TENTATIVE finalists to RESEARCH-LOG.md. (2) `01-premarket` slimmed to lightweight 8:48 AM refresh: read tentative list, refresh overnight US/geopolitical/VIX, re-validate Gate A live, invalidate on red flags or gap-up/down, write final green-list. (3) Slow-moving data cached: breadth + 200-SMA refresh weekly (Sundays), Gate C quality cached 7 days. (4) Pre-Gate-B candidate cap of 25 by relative-strength proxy. (5) Per-cron model assignment: Haiku for ops crons (auto-resume, midday, eod-Mon-Thu), Sonnet for research/decision crons. | Mon Apr 27 morning: v3.3 pre-market timed out — 200+ tool calls in one 30-min morning window blew the Claude Code usage budget before Gate D/E or git commit. Architectural mistake: cramming all work into morning. Fix: split work across two cron-windows + cache slow-moving data + smaller candidate pool + cheaper models for ops. Total daily token usage projected ~80% lower than v3.3. No strategy logic weakened — same Gates A→E + Stage F, just executed in two passes. |
