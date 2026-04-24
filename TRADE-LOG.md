# AAYA v3 — Trade Log

Every trade (entry + exit) and every daily EOD snapshot is logged here.
This file is append-only — never edit history.

---

## FORMAT

### Trade entry
```
### 2026-MM-DD HH:MM — ENTRY [STOCK]
- Shares: N @ Rs XXX
- Capital deployed: Rs XX,XXX
- GTT upper leg: Rs XXX (target +4%)
- GTT lower leg: Rs XXX (stop, ATR-based)
- Thesis: [one line — why this trade]
- Research-log reference: [date]
- Order ID: [kite order id]
```

### Trade exit
```
### 2026-MM-DD HH:MM — EXIT [STOCK]
- Shares: N @ Rs XXX
- Exit reason: GTT upper / GTT lower / Time stop / Emergency / Manual
- Gross P&L: +/- Rs XXX (+/- X.X%)
- Costs (brokerage + STT + stamp + sebi): Rs XXX
- STCG tax provision (20%): Rs XXX
- Net P&L: +/- Rs XXX (+/- X.X%)
- Hold duration: N days
- Win/Loss: W / L
- Lesson: [one line learning]
```

### Daily EOD snapshot
```
### 2026-MM-DD EOD
- Portfolio value: Rs X,XX,XXX
- Cash: Rs XX,XXX
- Positions: [STOCK1 +X.X%], [STOCK2 -X.X%]
- Nifty 50 close: XX,XXX (+/- X.XX%)
- VIX: XX.X
- Day P&L: +/- Rs XXX
- Week-to-date P&L: +/- Rs XXX
```

---

## TRADES

*(First trade will be logged here when placed)*

---

## EOD SNAPSHOTS

*(First EOD snapshot will be logged here Monday 3:10 PM)*
