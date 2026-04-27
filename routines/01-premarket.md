# 01 — Pre-Market Refresh (8:45 AM IST, Mon-Fri) — v3.4 LIGHTWEIGHT

You are running the AAYA v3.4 PRE-MARKET cron at 8:48 AM IST. **Most heavy work is already done by the 22:00 IST night-research cron the previous evening.** Your job is just the overnight delta + final regime check.

## YOUR ROLE

Read last night's TENTATIVE finalists. Refresh overnight US/geopolitical context. Re-validate Gate A regime. Invalidate any tentative pick on overnight red flags or gap-up/down. Write the FINAL trade plan + green-list seed for the 9:03 AM deep-research cron.

## HARD RULES

- **DO NOT place any orders** — that's 9:23 AM cron's job.
- **DO NOT re-run Gates B/C/D from scratch.** Trust last night's work; only invalidate.
- **DO NOT use Perplexity.** WebSearch + WebFetch + Kite MCP only.
- **moneycontrol.com BLOCKED** — use trendlyne / nseindia / investing.com / chartink / screener.in.
- If last night's RESEARCH-LOG.md entry is missing or shows "no setup tonight" → exit silently.
- If Kite session expired: Slack re-auth URL, exit. 9:03 cron will retry.

## STEP-BY-STEP

### Step 0 — Trading-day check
```
Read: /Users/rakesh/AAYA Calculation/aaya-v3/data/nse-holidays-2026.txt
If today's date is in the file → exit silently.
```

### Step 1 — Read last night's TENTATIVE list
```
Read: /Users/rakesh/AAYA Calculation/aaya-v3/RESEARCH-LOG.md
Find the most recent "TENTATIVE Pre-Market" entry — should be from last night ~22:05 IST.
```

If no tentative entry from last night → 00-night-research didn't run. Slack a warning, exit. Don't try to do its job in the morning window.

If tentative entry says "no setup tonight" → exit silently.

If tentative entry has 1+ finalists → proceed.

### Step 2 — Kite session
```
Call: mcp__kite__get_profile
If error → Slack re-auth URL, exit.
```

### Step 3 — Overnight refresh (lightweight, parallel)

WebSearch in parallel — these are cheap, single-fact lookups:
- `"Dow Jones close yesterday"` and `"S&P 500 close yesterday"`
- `"India VIX live today"`
- `"SGX Nifty futures live today"`
- `"Brent crude oil price today"`
- `"FII DII data NSE yesterday"` (yesterday's settled numbers)

Geopolitical delta — only if there's a "live" theme tracked from yesterday's night-research:
- WebSearch one targeted query per active theme: `"Hormuz news today"` / `"Russia Ukraine update"` etc.

Total: ~6–8 WebSearches. Should take 1–2 min.

### Step 4 — Re-validate Gate A regime (live)

```
A1 PANIC GATE:
  vix_live = (parsed from Step 3 search)
  If vix_live ≥ 22 → 🛑 PANIC. Drop ALL tentative picks. Slack panic note. Exit.

A2 BREADTH (use cached from night-research, no new compute):
  Read data/breadth-cache.json → breadth_pct
  Mode determined per v3.3 decision matrix.
```

If mode changed since last night (e.g., VIX spiked from 18 to 21):
- Re-rank: drop picks below new min_d_score
- Reduce: truncate to new max_positions

### Step 5 — Per-stock invalidation checks

For each tentative finalist:

**5a. Overnight news red flag** — quick WebSearch per stock (1 query):
```
WebSearch: "[STOCK] news overnight"
Drop if: SEBI/RBI/IT investigation announced, promoter exit/arrest, auditor resigned,
  earnings unexpectedly announced for today, major loss/litigation
```

**5b. Gap projection** — get current LTP via batch quotes:
```
mcp__kite__get_quotes for all tentative tickers in ONE call
For each:
  prev_close = (from last night's data in RESEARCH-LOG)
  pre_open_ltp = current LTP
  gap_pct = (pre_open_ltp - prev_close) / prev_close
  Drop if gap_pct > 0.02 (gap-up chase rule)
  Drop if gap_pct < -0.01 (knife guard)
```

### Step 6 — Final ordering

After invalidations, surviving picks form the **GREEN-LIST FOR 9:03 AM EXECUTION**. Truncate to top `max_positions` by D-score (carried over from night-research).

Apply Gate E final pass:
- E1 same-sector cap (max 2)
- E3 skip held positions
- E4 skip too-expensive

### Step 7 — Update RESEARCH-LOG.md

Edit today's entry to add a "## MORNING REFRESH (8:48 AM)" subsection BEFORE the tentative list:

```markdown
## YYYY-MM-DD — MORNING REFRESH (08:48 IST)

**Live regime check:**
- VIX live: 18.30 → A1 ✅
- Breadth (cached): 40% → A2 ✅
- Mode: 🟢 NORMAL → max 3, min D 8

**Overnight context:**
- Dow: -0.5%, S&P: -0.6% (mild risk-off)
- SGX Nifty: 24,100 (-0.3%)
- Brent: $87/bbl (Hormuz risk steady)
- FII yesterday: -₹2,500 Cr | DII: +₹1,800 Cr
- Geopolitical: Hormuz steady, no new flares

**Tentative → Final:**
- ACUTAAS: KEEP (no overnight news, gap +0.4%)
- BEL: DROP — earnings announced for today (catalyst gate)
- NATIONALUM: KEEP (no news, gap -0.2% within band)

**GREEN-LIST FOR 9:03 AM EXECUTION:**
1. ACUTAAS — entry 2,395 / stop 2,302 / T1 2,491 / 13 sh
2. NATIONALUM — entry 196 / stop 188 / T1 204 / 168 sh
```

### Step 8 — Slack (only if action)

```
Send only if final green-list non-empty OR a tentative pick was dropped:
🌅 AAYA v3.4 Morning Refresh
Mode: 🟢 NORMAL
Green-list (N): [STOCK1, STOCK2]
Dropped: [STOCK3 — reason]
9:03 AM cron picks this up.
```

If green-list empty AND no drops → silent.

### Step 9 — Commit + push
```
git add -A && git commit -m "premarket refresh YYYY-MM-DD" && git push origin main
```

## OUTPUT CONTRACT
- Tentative finalists read: N
- Dropped: N (reasons)
- Green-list count: N
- Mode: NORMAL/REDUCED/PANIC
- Slack sent: yes/no
- Tool calls used: ~15-25 (target)

## FAILURE MODES

| Failure | Response |
|---|---|
| Last night's RESEARCH-LOG missing | Slack alert, exit. Don't do night work in morning window. |
| Cache stale | Use last value, log warning, continue. |
| Kite expired | Slack re-auth, exit. |
| WebSearch rate-limited | Use whatever finished, conservative bias toward DROP. |
