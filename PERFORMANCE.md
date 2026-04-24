# AAYA v3 — Performance Tracker

The edge meter. Updated every Friday by the weekly-review cron.
This is the file that tells us whether to keep trading or stop.

---

## HEADLINE STATS (running)

| Metric | Value |
|---|---|
| Trades taken | 0 |
| Wins | 0 |
| Losses | 0 |
| **Win rate** | — |
| Total gross P&L | Rs 0 |
| Total costs + tax paid | Rs 0 |
| **Total net P&L** | Rs 0 |
| Starting capital | Rs 1,00,000 |
| Current capital | Rs 1,00,000 |
| **Return since start** | 0.00% |
| Nifty 50 return since start | — |
| **Alpha vs Nifty** | — |

---

## WIN RATE GATES (decision thresholds)

| After 20 trades | Win Rate | Action |
|---|---|---|
| < 43% | Stop trading, full review of thesis | 🚨 RED |
| 43-50% | Reduce size by 50%, keep testing | ⚠️ AMBER |
| 50-55% | Continue at current size | 🟡 YELLOW |
| 55-60% | Healthy, consider scaling | 🟢 GREEN |
| > 60% | Scale capital to Rs 2L | 🚀 GREEN+ |

---

## WEEKLY SUMMARIES (newest first)

### TEMPLATE
```
## Week ending YYYY-MM-DD

- Trades this week: N (W: X, L: Y)
- Win rate this week: XX.X%
- Gross P&L: +/- Rs XXX
- Net P&L (after tax + costs): +/- Rs XXX
- Best trade: [STOCK] +X.X% — [reason]
- Worst trade: [STOCK] -X.X% — [reason]
- Nifty return this week: +/- X.X%
- Our return this week: +/- X.X%
- Alpha this week: +/- X.X%

**Lessons:**
- [one line from winners]
- [one line from losers]

**Rule changes proposed:**
- [None / specific rule change with rationale]

**Next week plan:**
- [any change in sector focus, watchlist, or parameters]
```

---

## CUMULATIVE CHART (updated monthly)

```
Month       | Trades | Win% | Net P&L    | Cumulative | vs Nifty
------------|--------|------|------------|------------|----------
2026-04     | 0      | —    | Rs 0       | Rs 1,00,000| —
```

---

## LESSONS LIBRARY

A running list of things proven TRUE or FALSE in live trading.
Update this section with evidence, not opinions.

### PROVEN TRUE
*(Add lessons here with trade-ID evidence as they accumulate)*

### PROVEN FALSE
*(Add discarded rules here with trade-ID evidence)*

### OPEN QUESTIONS
- Does CDSL breakout signal work better than bank breakouts?
- Is 4% target too tight for monopoly stocks (might run to 8%)?
- Should we exit earlier on Thursday instead of Friday?
