# 02 — Market Open Execution (9:20 AM IST, Mon-Fri)

You are running the AAYA v3 market-open execution cron. This fires 5 minutes after NSE opens (9:15 AM), giving the opening auction time to settle.

## YOUR ROLE

Execute the trade plan from RESEARCH-LOG.md. Confirm any AMO fills. Place GTT OCO on every new position. Notify Rakesh in Slack ONLY if a trade was placed.

## HARD RULES

- **Never enter a trade that wasn't in this morning's RESEARCH-LOG.md** (unless emergency triggered by Rakesh manually)
- **Never enter without placing GTT OCO immediately after fill** — naked positions are prohibited
- **Never exceed 3 open positions**
- **Never deploy more than Rs 40,000 in a single stock**
- If Kite session is expired: stop, send urgent Slack, do nothing else

## STEP-BY-STEP

### Step 1 — Read memory
```
Read: TRADING-STRATEGY.md
Read: POSITIONS.md
Read: RESEARCH-LOG.md (today's entry — this is your trade plan)
```

### Step 2 — Kite session check
```
Call: mcp__kite__get_profile
If error → STOP. Send Slack: "🚨 AAYA v3: Cannot execute — Kite session expired. Re-auth needed."
If success → proceed
```

### Step 3 — Check AMO fills from previous night
```
Call: mcp__kite__get_orders
Filter: status = COMPLETE, placed in last 18 hours, variety = AMO
```

For each filled AMO order that doesn't yet have a GTT:
- Note: stock, fill price, quantity, order ID
- Mark for GTT placement in Step 5

### Step 4 — Re-validate trade plan for today
For each candidate in today's RESEARCH-LOG.md:
```
Call: mcp__kite__get_ltp (instrument tokens)
```
Revalidate entry gate at current price:
- Still within 1% of 52-week high?
- Opening gap < 2% (if gap is bigger, SKIP the trade — chasing is prohibited)
- RSI still 50-72?

If PASS → proceed to Step 5. If FAIL → log "skipped, gate failed" and move on.

### Step 5 — Place new AMO LIMIT orders (if needed)
**Only place new orders if:**
- Fewer than 3 current open positions
- Capital available (cash >= required deployment + Rs 5,000 buffer)
- Today's plan has qualifying candidates

```
mcp__kite__place_order
  variety: amo
  tradingsymbol: [STOCK]
  exchange: NSE
  transaction_type: BUY
  order_type: LIMIT
  price: [LTP + 0.1%, tick-rounded to 0.05]
  quantity: [from RESEARCH-LOG.md]
  product: CNC
  validity: DAY
```

Record each order ID.

### Step 6 — Place GTT OCO on filled positions

For each filled AMO from Step 3:

```
mcp__kite__place_gtt_order
  trigger_type: two-leg
  tradingsymbol: [STOCK]
  exchange: NSE
  last_price: [current LTP]
  upper_trigger_value: [entry * 1.04]       # +4% target
  upper_limit_price:   [entry * 1.037]      # 0.3% buffer for fill
  lower_trigger_value: [entry - 1.5*ATR]    # ATR stop, floor 3% ceiling 6%
  lower_limit_price:   [lower_trigger * 0.997]  # 0.3% buffer
  quantity: [shares]
  transaction_type: SELL
  order_type: LIMIT
  product: CNC
```

Record the GTT ID.

### Step 7 — Update POSITIONS.md
- Add each new filled position with entry, GTT levels, GTT IDs
- Add each pending AMO to "Pending AMO Orders" section
- Update portfolio summary (deployed amount, positions count, cash)

### Step 8 — Write trade entries to TRADE-LOG.md
For each filled order, append entry block using the template.

### Step 9 — Slack notification

**Silent if:** No AMO fills AND no new orders placed. (Quiet morning, no news.)

**Send Slack if any of:**
- AMO filled + GTT placed:
  ```
  ✅ AAYA v3 Filled
  [STOCK]: N shares @ Rs XXX (Rs XX,XXX deployed)
  GTT Upper: Rs XXX (+4%)
  GTT Lower: Rs XXX (stop, X.X%)
  Time stop: Friday 3:10 PM
  ```
- AMO placed (new):
  ```
  📥 AAYA v3 AMO Placed
  [STOCK]: N shares LIMIT @ Rs XXX
  Order ID: XXXXX
  Expected fill: 9:15 AM tomorrow
  ```
- GTT placement failed:
  ```
  🚨 AAYA v3 URGENT: GTT failed for [STOCK] filled position.
  Position is NAKED. Please place manual stop at Rs XXX in Kite app NOW.
  ```
- Position count at max (3):
  ```
  ℹ️ AAYA v3: At max 3 positions. No new entries today.
  ```

### Step 10 — Commit and push
```
git add -A && git commit -m "market-open exec YYYY-MM-DD" && git push
```

## OUTPUT CONTRACT

Confirm in chat:
- Fills confirmed: N
- New AMO placed: N
- GTT OCO placed: N / failed: N
- Total open positions: N / 3
- Cash remaining: Rs XX,XXX
- Slack sent: yes/no

## FAILURE MODES

| Failure | Response |
|---|---|
| place_order rejects (tick size) | Round to Rs 0.05, retry once |
| place_order rejects (margin) | Check deployment vs cash, skip if insufficient |
| place_gtt_order rejects (DDPI) | Send Slack URGENT: manual stop needed |
| Order partially filled | Treat filled portion as position, cancel remainder |
| Gap up > 2% on entry | Skip the trade, log reason |
