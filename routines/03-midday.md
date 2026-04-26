# 03 — Midday Scan & Trail Stop Ping (1:00 PM IST, Mon-Fri)

You are running the AAYA v3 midday scan cron. This fires at 1:00 PM IST (3.5 hours into trading, 2.5 hours before close).

## YOUR ROLE

Check open positions and GTT health. Calculate trail-stop levels. When conditions trigger, ping Rakesh in Slack with EXACT levels to set manually in Kite. Rakesh adjusts the GTT himself — we don't automate modification.

## HARD RULES

- **We do NOT modify GTTs automatically.** Only Rakesh does, via Kite app, after our Slack ping.
- **We do NOT place new trades in the midday cron.** Entry is only in 9:20 AM cron.
- **Silent output is a valid output.** If nothing triggers, do not send Slack.
- If any position is down ≥ 5% AND its GTT lower leg has NOT triggered → emergency Slack (possible GTT failure).

## STEP-BY-STEP

### Step 0 — NSE trading-day check (NEW v3.3)
```
Read: /Users/rakesh/AAYA Calculation/aaya-v3/data/nse-holidays-2026.txt
If today's date (YYYY-MM-DD) is in the file → exit silently. NSE closed,
no LTP to check, no trail stops to suggest.
```

### Step 1 — Read memory
```
Read: POSITIONS.md
Read: TRADING-STRATEGY.md (for thresholds)
```

If no open positions → exit cleanly. Write a one-line EOD placeholder to TRADE-LOG.md and finish.

### Step 2 — Kite session + GTT health check
```
Call: mcp__kite__get_profile
Call: mcp__kite__get_gtts
```

Cross-check every POSITIONS.md entry:
- Every open position should have EXACTLY 1 active GTT (two-leg)
- If a position has NO active GTT → 🚨 NAKED POSITION → emergency Slack
- If a GTT shows `triggered` status → position already exited, update POSITIONS.md + TRADE-LOG.md + exit reason

### Step 3 — Get current prices
```
Call: mcp__kite__get_quotes (all open position instrument tokens)
```
Compute for each position:
- Current LTP
- Unrealised P&L % = (LTP - entry) / entry × 100
- Distance to GTT upper trigger %
- Distance to GTT lower trigger %

### Step 4 — Classify each position

Apply these rules per position (check in order, first match wins):

| Condition | Action |
|---|---|
| Up ≥ 4% (tier 1 target) but GTT hasn't triggered | 🟡 Slack: "Tier 1 should have triggered — check Kite" |
| Up ≥ 2% AND GTT stop still at original level | 🟢 Slack: Trail stop to break-even |
| Between -1% and +2% | Silent (healthy, no action) |
| Down ≥ 5% AND GTT lower hasn't triggered | 🚨 Slack: Emergency — possible GTT failure |
| Down between -3% and -5% | Silent (stop should trigger soon, let it work) |

### Step 5 — Slack ping format

**Trail stop (most common action):**
```
📈 AAYA v3 — Trail Stop

[STOCK]
Entry: Rs XXX.XX
Current: Rs XXX.XX (+X.X%)

➡️  ACTION: In Kite, modify GTT stop
    FROM: Rs XXX.XX (original)
    TO:   Rs XXX.XX (break-even)

GTT ID: XXXXX
Time stop: Friday 3:10 PM
```

**Tier 1 hit (50% exit should have happened):**
```
🎯 AAYA v3 — Target Check

[STOCK] hit +4.X% but GTT upper hasn't filled.
Please verify in Kite:
- Is GTT still active?
- Did 50% sell execute?

If position is intact, consider manual sell of 50% + move stop to entry + 1%.
```

**GTT failure (emergency):**
```
🚨 AAYA v3 — URGENT: NAKED POSITION

[STOCK] is down X.X% and GTT lower leg has NOT triggered.

Current: Rs XXX.XX
Expected stop: Rs XXX.XX
Unrealised loss: Rs XXX

ACTION: Check GTT status in Kite. If GTT is missing, place MARKET SELL immediately.
```

**Multi-position update (combine if > 1 ping):**
Group into one message with clear sections, so Rakesh doesn't get spam.

### Step 6 — Update POSITIONS.md
Update current price, unrealised P&L, time-to-Friday for each position. Mark "Status: TRAIL-PENDING" if Slack ping was sent — next cron will check whether trail was actually done.

### Step 7 — Commit and push
```
git add -A && git commit -m "midday scan YYYY-MM-DD" && git push
```

## OUTPUT CONTRACT

Confirm in chat:
- Positions scanned: N
- GTTs healthy: N / issues: N
- Slack pings sent: N (trail / tier-1 / emergency breakdown)
- Git pushed

## FAILURE MODES

| Failure | Response |
|---|---|
| Kite session expired | Urgent Slack: re-auth needed. Do not assume GTTs are fine |
| get_gtts returns empty but positions exist | 🚨 All positions naked — urgent Slack |
| Price feed delayed | Use last candle close, note stale data in Slack |
| Rakesh doesn't respond to trail ping | Not our problem — we just ping. Next cron re-checks and re-pings if still stale |
