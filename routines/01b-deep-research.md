# 01b — Deep Research Red-Flag Scan (9:00 AM IST, Mon-Fri)

You are running the AAYA v3.3 deep-research cron. This fires between the 8:48 AM pre-market cron and the 9:23 AM market-open cron.

## YOUR ROLE

For each candidate produced by `01-premarket.md` Gates A→E, do per-stock deep research to catch hard red flags that the quantitative filter cannot see (SEBI actions, promoter exits, earnings dates, retail mania). Drop any candidate with a hard red flag BEFORE the 9:23 AM market-open cron tries to place an order.

Better to occasionally drop a valid setup than to enter a stock with a known red flag.

## HARD RULES

- **DO NOT add stocks** to the candidate list. You can only DROP. The filter decided the universe; you only veto.
- **DO NOT place orders.** That's 02-market-open's job at 9:23 AM.
- **DO NOT use Perplexity or any paid API.** WebSearch + WebFetch + Kite MCP only.
- **DO NOT use moneycontrol.com** — blocked. Use nseindia / trendlyne / investing.com / chartink / screener.in.
- If RESEARCH-LOG.md has zero candidates today (Gate A panic, or 0 finalists) → exit silently.

## STEP-BY-STEP

### Step 0 — Trading day + readiness gate (NEW v3.3 fix)

**0a. NSE holiday check.**
```
Read: /Users/rakesh/AAYA Calculation/aaya-v3/data/nse-holidays-2026.txt
If today's date (YYYY-MM-DD) is in the file → exit silently, no Slack.
```

**0b. Wait for 01-premarket to finish (readiness probe).**
01-premarket fires at 8:48 AM and may take 5–15 min for a full Gates A→E scan. We must not run on stale data.
```
Read RESEARCH-LOG.md
Look for an entry dated today (YYYY-MM-DD) with a "Trade ideas" or "Finalists" or
  "ACTION FOR 9:20 AM CRON" / "GREEN-LIST FOR 9:23 AM" section produced TODAY.

If today's entry exists AND has finalists → proceed.
If today's entry exists AND says "Gate A panic" or "0 picks today" → exit silently, no work to do.
If today's entry does NOT exist yet:
  - Wait 60 seconds
  - Re-read RESEARCH-LOG.md
  - Repeat up to 3 times (total wait: ~3 min)
  - If still missing after 3 retries:
      Slack to D0AQCRLP7SP: "⚠️ AAYA v3.3: 01b deep-research fired but
        01-premarket has not produced today's entry. Skipping deep-research."
      Exit.
```

**0c. Idempotency check.**
```
If today's RESEARCH-LOG.md entry ALREADY contains a "## DEEP RESEARCH RESULTS" subsection:
  → Already ran today (manual re-run, retry, or auto-resume). Exit silently.
  → Do NOT re-process candidates; the green-list is final.
```

### Step 1 — Read state
```
Read: /Users/rakesh/AAYA Calculation/aaya-v3/RESEARCH-LOG.md (today's entry — has finalists)
Read: /Users/rakesh/AAYA Calculation/aaya-v3/TRADING-STRATEGY.md (Stage F definitions)
```

Identify the candidate list. If empty → exit, no Slack.

### Step 2 — Kite session check
```
Call: mcp__kite__get_profile
If error → Slack D0AQCRLP7SP "⚠️ AAYA v3.3 deep-research: Kite expired. Login URL → [from mcp__kite__login]" → exit. The 9:23 cron will repeat the warning.
```

### Step 3 — Per-stock red-flag scan

For EACH candidate from RESEARCH-LOG.md, run ALL of the following checks IN PARALLEL where possible:

#### F1. NEWS SCAN (last 30 days)
```
WebSearch: "[STOCK] news"
WebSearch: "[STOCK] NSE announcement 2026"
WebSearch: "[STOCK] SEBI" (separate query — surface regulatory hits fast)
```

**HARD RED FLAGS (drop the stock):**
- SEBI / RBI / income-tax / ED / CBI investigation announced
- Promoter resignation, arrest, or detention
- Auditor resignation or qualification (going-concern, material misstatement)
- Material loss / accident / fire / litigation > Rs 100 Cr
- Major customer loss (>20% of revenue mentioned in news)
- Credit rating downgrade by CRISIL/ICRA/CARE/Fitch/Moody's

**Soft notes (log, don't drop):**
- Sector tailwind announcement (e.g., defence order, PSU disinvestment)
- Positive guidance reiterated

#### F2. CATALYST CALENDAR
```
WebSearch: "[STOCK] earnings date 2026 Q4"
WebSearch: "[STOCK] ex dividend date"
WebSearch: "[STOCK] AGM date 2026"
WebSearch: "[STOCK] board meeting"
```

**HARD RED FLAGS (drop):**
- Earnings call within next 5 TRADING DAYS — too much gap risk on a 1-week hold
- Ex-dividend within next 3 trading days — price drop already priced in, distorts our entry math
- AGM within 2 days — uncertainty premium

#### F3. INSIDER ACTIVITY (last 30 days)
```
WebFetch: https://www.nseindia.com/companies-listing/corporate-filings-insider-trading
  (filter by symbol = [STOCK])
WebSearch: "[STOCK] insider trading promoter selling 2026"
WebSearch: "[STOCK] promoter pledge increase"
```

**HARD RED FLAGS (drop):**
- Promoter or designated person SOLD > 0.5% of total float in last 30 days
- Promoter pledge INCREASED (any amount) in last 30 days

**Soft signals (note in log, don't drop):**
- Promoter BUYING in last 90 days → +1 conviction note (already a Gate D8 bonus)

#### F4. RETAIL MANIA / TOP-SIGNAL CHECK
```
WebSearch: "[STOCK] stock twitter today"
WebSearch: "[STOCK] multibagger 2026"
WebSearch: "[STOCK] target 2x"
```

**HARD RED FLAG (drop):**
- Multiple viral threads / videos calling it a "multibagger" or "100% target"
- Stock trending in retail discourse with parabolic price action (already up >20% this month)
- "Sure shot" / "guaranteed" / "operator-backed" language in retail circles

Reasoning: when retail piles in late, institutions distribute. Parabolic retail mania is a top signal, not an entry signal.

#### F5. ANALYST SANITY CHECK (soft)
```
WebSearch: "[STOCK] brokerage target price 2026"
WebSearch: "[STOCK] upgrade downgrade analyst"
```

**Don't drop on this check.** Just record:
- Any upgrade in last 30 days → log "+ analyst upgrade [broker] target Rs XXX"
- Any downgrade in last 30 days → log "- analyst downgrade [broker]" (note but don't drop — analysts lag price)

#### F6. SECTOR ROTATION CHECK
Cross-reference today's sector heatmap (from 01-premarket.md Step 5) with 90-day-ago sector ranking.

```
WebSearch: "Nifty sector performance January 2026"
WebSearch: "Nifty sector performance last 3 months"
```

**HARD RED FLAG (drop):**
- Stock's sector was in top-3 performers 90 days ago AND is in bottom-3 today
- Clear rotation OUT — momentum decay regime for the entire theme

### Step 4 — Per-stock decision
For each candidate, compile a single dossier:

```
[STOCK] — [GREEN / RED]

F1 News:           [PASS / RED: <reason>]
F2 Catalyst:       [PASS / RED: earnings on <date>]
F3 Insider:        [PASS / RED: promoter sold X% on <date>]
F4 Retail:         [PASS / RED: viral mania detected]
F5 Analyst:        [+upgrade / -downgrade / neutral]  (informational)
F6 Sector:         [PASS / RED: sector rotated out]

Verdict: [GREEN / RED]
If RED, primary reason: [...]
Conviction note (if GREEN): [...optional positive context...]
```

### Step 5 — Update RESEARCH-LOG.md
Edit today's entry to add a "## DEEP RESEARCH RESULTS" subsection BEFORE the trade plan:

```
## DEEP RESEARCH RESULTS (9:00 AM scan)

| Stock | F1 News | F2 Catalyst | F3 Insider | F4 Retail | F6 Sector | Verdict |
|---|---|---|---|---|---|---|
| ACUTAAS | PASS | PASS | PASS | PASS | PASS | 🟢 GREEN |
| BEL | PASS | RED ❌ earnings May 2 | – | – | – | 🔴 RED — drop |
| NATIONALUM | PASS | PASS | PASS | PASS | PASS | 🟢 GREEN |

DROPPED: BEL (earnings within 5 days)
GREEN-LIST FOR 9:23 AM EXECUTION: ACUTAAS, NATIONALUM
```

The 02-market-open cron MUST honor the green-list. Any RED stock from this stage will not have an order placed.

### Step 6 — Slack notification (only if drops occurred)

**Silent if:** all candidates pass (GREEN). 02-market-open will Slack on actual fills.

**Send Slack if any drops:**
```
🔍 AAYA v3.3 Deep-Research Drops
[N] candidate(s) dropped before market open:
• [STOCK]: [primary reason — e.g. "earnings May 2"]
• [STOCK]: [...]
GREEN-list for 9:23 AM: [stocks] (or "none — no entries today")
```

### Step 7 — Commit and push
```
cd "/Users/rakesh/AAYA Calculation/aaya-v3"
git add -A && git commit -m "deep research YYYY-MM-DD" && git push origin main
```

## OUTPUT CONTRACT

Confirm in chat:
- Candidates evaluated: N
- Dropped: N (with reasons)
- Green-list passed to 02-market-open: [stocks]
- Slack sent: yes/no
- Git pushed: yes/no

## FAILURE MODES

| Failure | Response |
|---|---|
| WebSearch rate-limited mid-scan | Continue with whatever finished; log incomplete checks as "PASS (skipped)"; conservative — don't drop on missing data |
| Kite session expired | Slack expiry note; do NOT drop anyone (let 02-market-open handle re-auth) |
| 0 candidates from 01-premarket | Exit silently, no Slack |
| Conflicting news (one source says SEBI, another denies) | Default to DROP — false positive < false negative |
| Earnings date ambiguous | If earnings could be within 5 days based on quarterly calendar, DROP. Better safe. |
