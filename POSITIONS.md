# AAYA v3 — Current Positions

Live state of all open positions and pending orders.
Updated by every cron that changes state.

---

## PORTFOLIO SUMMARY

- **Total capital:** Rs 1,00,000
- **Deployed:** Rs 0 (0%)
- **Cash available:** Rs 1,00,000 (100%)
- **Open positions:** 0 / 3 max
- **Last updated:** 2026-04-24 (initial setup)

---

## OPEN POSITIONS

*(None yet. First position will appear here after market-open cron fills an order.)*

### TEMPLATE
```
### [STOCK]
- Shares: N
- Entry: Rs XXX (on YYYY-MM-DD HH:MM)
- Current: Rs XXX (as of last cron)
- Unrealised P&L: +/- Rs XXX (+/- X.X%)
- GTT Upper: Rs XXX (target +4%) — GTT ID: XXXXX
- GTT Lower: Rs XXX (stop) — GTT ID: XXXXX
- ATR(14) at entry: Rs X.XX
- Time to Friday exit: N days
- Thesis: [one line]
- Status: [HEALTHY / WATCHING / BREAK-EVEN-STOP / TRAIL-PENDING]
```

---

## PENDING AMO ORDERS

*(Orders placed after market hours, waiting for next session)*

### TEMPLATE
```
### [STOCK] — AMO LIMIT BUY
- Shares: N @ Rs XXX
- Placed: YYYY-MM-DD HH:MM
- Expected fill: next session open
- Order ID: XXXXX
- Plan: place GTT OCO immediately after fill confirmation
```

---

## HISTORICAL POSITIONS (last 30 days)

*(Closed positions move here after exit for quick reference)*

---

## LEGACY v1/v2 POSITIONS (from previous strategy)

**Note:** 4 positions from April 2026 v2 execution — pre-date v3 rotation strategy.

| Stock | Shares | Entry | Capital | Action under v3 |
|---|---|---|---|---|
| SBIN | 1 | ~Rs 770 | Rs 770 | Hold until 12-month LTCG mark (April 2027) unless fundamentally broken |
| POWERGRID | 1 | ~Rs 288 | Rs 288 | Hold until 12-month LTCG mark |
| NTPC | 1 | ~Rs 355 | Rs 355 | Hold until 12-month LTCG mark |
| COALINDIA | 1 | ~Rs 430 | Rs 430 | Hold until 12-month LTCG mark |

*Total capital locked: ~Rs 1,843 (negligible vs Rs 1L rotation budget). These are kept as long-term holdings, separate from the v3 weekly rotation pool. They do NOT count toward the 3-position rotation limit. If any of them later surfaces through the Nifty 500 momentum scan as a valid v3 entry, we simply don't double-up.*
