# AAYA v3.1 — Trading Strategy (The Rulebook)

**Last updated:** 2026-04-24
**Version:** 3.1 (post-review filter tune)
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

## ENTRY PIPELINE — Gates A → E (ALL must pass)

### Gate A — Market Regime (daily check, binary)
```
A1. Nifty 500 close > Nifty 500 200-day SMA
A2. India VIX < 22
```
If A fails → zero trades today. Skip the rest of the pipeline.

---

### Gate B — Stock Momentum (6 conditions, all must pass)
```
B1. (high_52w - close) / high_52w ≤ 0.05                        # within 5% of 52-wk high
B2. count(sessions where volume ≥ 1.5 × vol_avg_20,
          over last 5 sessions) ≥ 3                              # sustained volume, not one-off
B3. 50 ≤ RSI(14) ≤ 80                                            # momentum zone, wider to allow trending
B4. close > SMA(50)  AND  SMA(50) > SMA(50) ten sessions ago     # 50-SMA rising
B5. close > SMA(200)                                             # stock in long-term uptrend
B6. daily_traded_value ≥ Rs 50,00,000                            # liquidity floor
```

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

### ORDER TYPE
AMO LIMIT at LTP + 0.1%, tick-rounded to Rs 0.05. Never market orders at entry (except re-entry within 30 min of fill if AMO was rejected).

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
