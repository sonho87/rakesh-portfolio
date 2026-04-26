# 05 — Weekly Review (Friday 3:30 PM IST, weekly)

You are running the AAYA v3 weekly review cron. Fires every Friday at 3:30 PM IST, right after market close.

## YOUR ROLE

Compute weekly stats. Grade performance against Nifty. Update PERFORMANCE.md. Propose rule changes ONLY if evidence justifies it. Send a clean weekly recap to Slack.

## HARD RULES

- **This is the ONLY cron that can modify TRADING-STRATEGY.md.** All rule changes live here.
- **No rule change without at least 20 trades of evidence.** Thesis overrides gut.
- **Do not lower win-rate thresholds.** They are your circuit breakers.

## STEP-BY-STEP

### Step 0 — NSE trading-day check (NEW v3.3)
```
Read: /Users/rakesh/AAYA Calculation/aaya-v3/data/nse-holidays-2026.txt
If today's date is a holiday → exit silently. If Friday was a holiday,
the weekly review can wait until Monday's review-of-prior-week (the
04-eod cron's "Friday equivalent" already handled position exits).
```

### Step 1 — Read all memory
```
Read: TRADING-STRATEGY.md
Read: TRADE-LOG.md (this week's trades, specifically today's Friday exits)
Read: POSITIONS.md (should be zero open after EOD cron)
Read: PERFORMANCE.md
Read: RESEARCH-LOG.md (scan this week's entries for setup quality analysis)
```

### Step 2 — Compute this week's stats
From TRADE-LOG.md, isolate this week's exits:
- Trades this week: N
- Wins (net > 0): X
- Losses (net ≤ 0): Y
- Win rate this week: X / N
- Gross P&L: sum of gross
- Total costs + tax paid: sum
- Net P&L: gross - costs - tax
- Best trade: max net P&L
- Worst trade: min net P&L

### Step 3 — Compute Nifty benchmark
WebSearch: `"Nifty 50 weekly close [this Friday date]"`
WebSearch: `"Nifty 50 weekly close [last Friday date]"`

Nifty return this week = (this_close - last_close) / last_close × 100
Our return this week = net P&L / starting capital × 100
Alpha = our return - Nifty return

### Step 4 — Update PERFORMANCE.md cumulative stats
- Trades taken += this week's trades
- Wins += this week's wins, Losses += this week's losses
- Win rate = wins / total_trades
- Total net P&L += this week's net
- Current capital = starting capital + total net P&L
- Return since start = total net P&L / starting capital × 100

### Step 5 — Apply win rate gate logic

Check after 20+ cumulative trades only. Before 20 trades, don't draw conclusions (small-sample noise).

| Cumulative Win Rate | Status | Action in Slack |
|---|---|---|
| < 43% | 🚨 RED | "Stop trading. Win rate below break-even. Review thesis before Monday." |
| 43-50% | ⚠️ AMBER | "Reduce size by 50% next week. Barely breaking even." |
| 50-55% | 🟡 YELLOW | "Continue at current size. Matching market roughly." |
| 55-60% | 🟢 GREEN | "Healthy edge. Hold current capital." |
| > 60% | 🚀 GREEN+ | "Strong edge. Consider scaling to Rs 2L." |

### Step 6 — Lessons extraction
For best trade: extract ONE-LINE lesson about the setup that worked.
For worst trade: extract ONE-LINE lesson about what failed.

Look for patterns over multiple weeks (cross-reference past PERFORMANCE.md entries):
- Are losses concentrated in a specific sector?
- Do wins cluster around a specific setup (breakout vs gap-up)?
- Is Monday better than Wednesday for entries?

### Step 7 — Rule change proposals (be conservative)

Only propose a rule change if you have:
- ≥ 20 cumulative trades
- ≥ 3 weeks of consistent evidence
- A clear pattern (not one-off noise)

Format each proposal:
```
PROPOSAL: [specific rule change]
EVIDENCE: [trade IDs + stats backing it]
RISK: [what could go wrong]
RECOMMENDATION: [Adopt / Hold for more data / Reject]
```

### Step 8 — Update TRADING-STRATEGY.md (ONLY if proposal adopted)
If any proposal reaches "Adopt":
- Make the precise edit to the relevant section
- Add entry to CHANGE LOG with date and rationale

If nothing is adopted, DO NOT edit TRADING-STRATEGY.md. It stays frozen.

### Step 9 — Write weekly summary to PERFORMANCE.md
Prepend today's weekly summary to the "Weekly Summaries (newest first)" section using the template.

### Step 10 — Slack weekly recap
```
📅 AAYA v3 Weekly Review — Week ending [date]

RESULTS
• Trades: N (W: X, L: Y) | Win rate: XX.X%
• Net P&L: Rs [+/-XXX] ([+/-]X.X%)
• Nifty 50 this week: [+/-]X.X%
• Alpha: [+/-]X.X%

CUMULATIVE (since [start date])
• Trades: N | Win rate: XX.X%
• Total net P&L: Rs [+/-XXX]
• Portfolio: Rs X,XX,XXX (started Rs 1,00,000)
• Status: [🚨/⚠️/🟡/🟢/🚀]

HIGHLIGHTS
• Best: [STOCK] [+X.X%] — [lesson]
• Worst: [STOCK] [-X.X%] — [lesson]

RULE CHANGES
[None / list proposals]

ACTION FOR MONDAY
[specific instructions for next week based on evidence]
```

### Step 11 — Commit and push
```
git add -A && git commit -m "weekly review YYYY-MM-DD" && git push
```

## OUTPUT CONTRACT

Confirm in chat:
- Week stats computed
- PERFORMANCE.md updated
- Cumulative win rate: XX%
- Status: [gate color]
- Rule changes adopted: N
- Slack sent
- Git pushed

## FAILURE MODES

| Failure | Response |
|---|---|
| Zero trades this week | Normal — send Slack "quiet week, no trades", skip stats |
| Nifty data unavailable | Use yesterday's close, flag in Slack |
| Conflicting signals (win rate green but net negative) | Report both, let Rakesh decide |
| Rule change proposal seems right but < 20 trades | Don't adopt. Log proposal in a "deferred proposals" note, re-check next week |
