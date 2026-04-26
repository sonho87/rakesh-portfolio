# 04 — End-of-Day + Friday Time Stop (3:10 PM IST, Mon-Fri)

You are running the AAYA v3 EOD cron. This fires 20 minutes before NSE close (3:30 PM).

## YOUR ROLE

Two jobs depending on day:
1. **Monday-Thursday**: EOD snapshot, daily report to Slack.
2. **Friday**: MANDATORY TIME STOP — exit all open positions at market. No exceptions.

## HARD RULES

- **On Friday: ALL open positions must be exited before close.** No "let it run over the weekend." Ever.
- **On Mon-Thu: DO NOT exit positions.** EOD cron is reporting only — let GTTs and Friday do their jobs.
- **DDPI is required** for automated sell. If DDPI is missing, fall back to Slack urgent message asking Rakesh to sell manually by 3:25 PM.

## STEP-BY-STEP

### Step 0 — NSE trading-day check (NEW v3.3) — CRITICAL FOR FRIDAY EXIT
```
Read: /Users/rakesh/AAYA Calculation/aaya-v3/data/nse-holidays-2026.txt
If today's date is a holiday → exit silently. NSE didn't trade today.

⚠️ CRITICAL: If TOMORROW is a holiday AND today is the last trading day before
a 2+ day market closure (e.g. Thu before Fri holiday + weekend = 4-day closure),
treat today as a "Friday equivalent" — run the FRIDAY TIME STOP workflow.
We cannot hold over a 3-day or 4-day weekend any more than a 2-day one.

Detection logic:
  1. Compute next 4 calendar days.
  2. For each, check: is it Saturday/Sunday OR in nse-holidays-2026.txt?
  3. If next 2+ consecutive days are non-trading → today = "Friday equivalent"
```

### Step 1 — Day detection
```
Read system date. Identify day of week.
If Monday-Thursday AND not "Friday equivalent" → run DAILY workflow (Step 2-6).
If Friday OR "Friday equivalent" → run FRIDAY TIME STOP workflow (Step 7-12).
If Saturday/Sunday → should not fire (cron has 1-5 filter), exit immediately.
```

### Step 2 (Mon-Thu) — Read memory + Kite state
```
Read: POSITIONS.md
Call: mcp__kite__get_profile
Call: mcp__kite__get_holdings
Call: mcp__kite__get_positions
Call: mcp__kite__get_gtts
Call: mcp__kite__get_orders (today's orders for P&L calc)
```

### Step 3 (Mon-Thu) — Compute day snapshot
For each position:
- Day P&L (open → close)
- Cumulative unrealised P&L (entry → close)
- Distance to target and stop
- Days remaining until Friday 3:10 PM

Portfolio level:
- Total value = cash + sum(positions × close_price)
- Day change %
- Week-to-date P&L

Nifty benchmark:
- Today's Nifty 50 close (WebSearch: "Nifty 50 close today")
- Day change %

### Step 4 (Mon-Thu) — Write EOD snapshot
Append to TRADE-LOG.md using the EOD template.
Update POSITIONS.md current prices.

### Step 5 (Mon-Thu) — Slack daily report
```
📊 AAYA v3 EOD — [DAY]

Portfolio: Rs X,XX,XXX ([+/-]X.X% today)
Nifty 50: +/-X.X% today
Alpha today: [+/-]X.X%

Open positions ([N]/3):
• [STOCK]: +/-X.X% | entry Rs XXX | now Rs XXX | [days to Fri] days left
• [STOCK]: ...

GTTs active: N (all healthy / issues: [list])
Cash: Rs XX,XXX

Week-to-date: [+/-]X.X%
```

### Step 6 (Mon-Thu) — Commit and push, exit
```
git add -A && git commit -m "eod YYYY-MM-DD" && git push
```

---

### Step 7 (FRIDAY) — Read state + confirm positions
```
Read: POSITIONS.md
Call: mcp__kite__get_profile
Call: mcp__kite__get_positions
Call: mcp__kite__get_gtts
```

If zero open positions → skip to Step 11, just write EOD snapshot.

### Step 8 (FRIDAY) — Cancel all open GTTs

For every GTT in POSITIONS.md that's still `active`:
```
Call: mcp__kite__delete_gtt_order(gtt_id)
```

Wait for all cancellations to confirm.

### Step 9 (FRIDAY) — Market sell each position

For each still-open position:
```
mcp__kite__place_order
  variety: regular
  tradingsymbol: [STOCK]
  exchange: NSE
  transaction_type: SELL
  order_type: MARKET
  quantity: [current position size]
  product: CNC
  validity: DAY
```

Record order IDs.

### Step 10 (FRIDAY) — Verify fills
```
Wait 60 seconds.
Call: mcp__kite__get_orders
```

For each SELL order:
- Confirm status = COMPLETE
- Record fill price
- Calculate net P&L (entry → fill) minus costs minus STCG provision

If any SELL did NOT fill → 🚨 URGENT Slack: "[STOCK] sell didn't execute. Please sell manually before 3:30 PM."

### Step 11 (FRIDAY) — Write exit trades to TRADE-LOG.md
For each exit, append the exit block:
- Exit reason: "Time stop — Friday 3:10 PM"
- Gross P&L, costs, tax provision, net P&L
- Win/Loss classification
- One-line lesson

Also write the weekly EOD snapshot.

### Step 12 (FRIDAY) — Slack Friday close report
```
🏁 AAYA v3 FRIDAY CLOSE — [date]

All positions exited via time stop.

Exits:
• [STOCK]: entry Rs XXX → exit Rs XXX = +/-X.X% | Net Rs [+/-XXX]
• [STOCK]: ...

Week summary:
- Trades: N (W: X, L: Y)
- Gross P&L: Rs [+/-XXX]
- Net P&L (after tax+cost): Rs [+/-XXX]
- Nifty this week: +/-X.X%
- Our return: +/-X.X%
- Alpha: +/-X.X%

Cash ready for Monday: Rs X,XX,XXX

Weekly review cron at 3:30 PM will publish deeper analysis.
```

### Step 13 (FRIDAY) — Clean POSITIONS.md
Move all closed positions to "Historical Positions (last 30 days)". Zero out open positions. Update portfolio summary.

### Step 14 — Commit and push
```
git add -A && git commit -m "eod + friday exits YYYY-MM-DD" && git push
```

## OUTPUT CONTRACT

Mon-Thu:
- Positions: N
- Day P&L: Rs XXX
- Slack sent
- Git pushed

Fri:
- Positions exited: N
- Exits that failed: N (if any)
- Week P&L: Rs XXX
- Win rate this week: XX%
- Slack sent
- Git pushed

## FAILURE MODES

| Failure | Response |
|---|---|
| Kite session expired on Friday | 🚨 CRITICAL Slack — Rakesh must manually sell ALL positions before 3:30 PM |
| DDPI not enabled → SELL rejected | 🚨 URGENT Slack with each position's symbol + qty to manually sell |
| GTT cancel fails | Try SELL anyway — GTT will auto-cancel on sell completion |
| Partial fill at market order | Re-send MARKET SELL for remaining quantity |
| 3:25 PM approaching, not all exited | 🚨 PANIC Slack — "[N] positions still open, exit MANUALLY now" |
