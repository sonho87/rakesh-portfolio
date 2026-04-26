# AAYA v3 — Current Positions

Live state of all open positions and pending orders.
Updated by every cron that changes state.

---

## PORTFOLIO SUMMARY

- **Total capital:** Rs 1,00,000
- **Deployed:** Rs 0 (0%)
- **Cash available:** Rs 1,00,000 (100%)
- **Open positions:** 0 / 3 max
- **Last updated:** 2026-04-26 (Kite holdings/positions reconciled — both empty, clean slate)
- **Gate A status (last check):** ❌ FAILED — Nifty 500 (22,570) below 200-SMA (23,120). No entries until regime turns.

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

**RECONCILIATION 2026-04-26:** `mcp__kite__get_holdings` returned empty array. The legacy SBIN/POWERGRID/NTPC/COALINDIA positions are **NOT in this Zerodha account (AMT521)**. Either previously sold, or held in a different broker. Removing from active tracking.

If they reappear in a future `get_holdings` call, restore the table. Until then, this section is informational only.

~~| SBIN | 1 | ~Rs 770 | Rs 770 | Hold until 12-month LTCG mark (April 2027) |~~
~~| POWERGRID | 1 | ~Rs 288 | Rs 288 | Hold until 12-month LTCG mark |~~
~~| NTPC | 1 | ~Rs 355 | Rs 355 | Hold until 12-month LTCG mark |~~
~~| COALINDIA | 1 | ~Rs 430 | Rs 430 | Hold until 12-month LTCG mark |~~
