# 02 — Market Open Execution (9:23 AM IST, Mon-Fri) — v3.3

You are running the AAYA v3.3 market-open execution cron. This fires after `01b-deep-research` (9:00 AM) so the green-list is already finalised.

## YOUR ROLE

Execute the green-list from RESEARCH-LOG.md (only candidates that passed Stage F deep research). Confirm any AMO fills from overnight. Place GTT OCO on every new position. Use the v3.3 hybrid entry — primary AMO LIMIT + intraday fallback if primary doesn't fill.

## HARD RULES

- **Only enter stocks on the GREEN-LIST from `01b-deep-research`'s scan today.** Any candidate marked RED or dropped is forbidden.
- **Never enter without placing GTT OCO immediately after fill** — naked positions are prohibited
- **Never exceed max_positions** from Gate A regime mode (2 in REDUCED, 3 in NORMAL)
- **Never deploy more than Rs 40,000 in a single stock**
- **Knife guard**: Never enter if stock is down >1% from prev close at order time
- **Chase guard**: Never enter if stock is up >2% from prev close (gap-up rule)
- **Regime flip guard**: If VIX has spiked to ≥22 since pre-market, cancel all pending and skip
- If Kite session is expired: stop, send urgent Slack, do nothing else

## STEP-BY-STEP

### Step 0 — NSE trading-day check (NEW v3.3)
```
Read: /Users/rakesh/AAYA Calculation/aaya-v3/data/nse-holidays-2026.txt
If today's date (YYYY-MM-DD) is in the file → exit silently. No Kite calls,
no orders. NSE is closed.
```

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

### Step 4 — Re-validate green-list at market open

Read the GREEN-LIST FOR 9:23 AM EXECUTION from today's RESEARCH-LOG.md (set by `01b-deep-research`). If no green-list section exists or it's empty → exit silently, no orders.

For each green-list candidate:
```
Call: mcp__kite__get_ltp / mcp__kite__get_quotes
```

Revalidate at current price:
- ✅ Still within 5% of 52-week high (B1)? — sanity check
- ✅ Opening gap < 2% above prev close — if gap bigger, SKIP (chase guard)
- ✅ Stock NOT down >1% from prev close — if it is, SKIP (knife guard, v3.3 NEW)
- ✅ Gate B7 still holds — re-compute (stock 30d return - Nifty 500 30d return) ≥ +5%
- ✅ VIX still < 22 — fetch via WebSearch / Kite quotes for VIX index (regime guard)

If ALL pass → place primary order in Step 5. If ANY fail → log "skipped, [reason]" and move to next.

### Step 5 — Place primary entry order (v3.3 hybrid step 1)

For each candidate that passed Step 4 revalidation, AND we still have headroom (open_count < max_positions, cash >= position_size + Rs 5,000 buffer):

```
mcp__kite__place_order
  variety: regular         # NOT amo — market is already open
  tradingsymbol: [STOCK]
  exchange: NSE
  transaction_type: BUY
  order_type: LIMIT
  price: [LTP + 0.1%, tick-rounded to 0.05]
  quantity: [from RESEARCH-LOG.md]
  product: CNC
  validity: DAY
```

Record each order ID with timestamp.

### Step 5b — Intraday fallback entry (v3.3 hybrid step 2)

**🚫 HARD SKIP — FRIDAY:** If today is Friday, skip Step 5b entirely. Primary fill on Friday or no entry. Reason: 04-eod cron at 3:11 PM force-exits everything Friday — a fallback fill at 9:30 AM with mandatory market-sell at 3:11 PM same day = guaranteed loss after 0.3% round-trip costs (~Rs 99 per Rs 33K trade). The thesis needs ≥1 full session.

After 7 minutes (~9:30 AM, Mon-Thu only), check the primary orders:
```
Call: mcp__kite__get_orders
For each primary order placed in Step 5:
  - If status == COMPLETE → great, move to Step 6 (GTT placement)
  - If status == OPEN/PENDING and ltp not yet hit limit:
       Cancel the primary order (mcp__kite__cancel_order)
       Then re-evaluate FALLBACK conditions:
         (a) Today NOT Friday  ←── NEW v3.3 fix
         (b) Stock NOT down >1% from prev close
         (c) Stock NOT up >2% from prev close
         (d) Gate B7 still holds at current LTP
         (e) VIX still < 22
       If ALL 5 pass:
         Place fresh regular LIMIT @ current LTP + 0.05%, validity DAY
         (only one fallback attempt; no further retries that day)
       Else:
         Log "fallback skipped: [reason]", no entry for this stock today.
```

Note: Only ONE fallback attempt per stock per day. If the fallback also doesn't fill, no entry today.

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
