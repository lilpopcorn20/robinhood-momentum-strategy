# Robinhood Momentum Strategy v1 (full rules for automated runs)

Aggressive Nasdaq + S&P 500 momentum strategy for the Robinhood **Agentic**
account (account_number `831820246`). Starting equity **$200.00** — all
risk %s are equity-relative and recompute automatically; "starting equity"
only resets when the user adds/removes funds or explicitly begins a new
tracking period (see `state.json`, do not assume it's still $200).

**Execution mode is Signal-Only by default** (`state.json.execution_mode`).
In Signal-Only, produce full order tickets but place **no live orders** —
the user executes manually. Never place a live order unless
`execution_mode` reads `"autonomous"` in `state.json` AND
`kill_switch_engaged` is `false`.

## Step 0 — every run, before anything else
1. Read `state.json` in this repo. If `kill_switch_engaged` is `true`,
   stop immediately — do not scan, do not call any Robinhood tool beyond
   what's needed to report status. Report to the user and end the run.
2. Fetch live account equity via `get_portfolio` (account `831820246`) —
   never reuse a cached figure from a previous run.
3. If `current_session_date` in `state.json` is not today: set
   `start_of_day_equity` to the live equity just fetched, set
   `current_session_date` to today, set `daily_loss_limit_hit_today` to
   `false`. Leave `consecutive_losing_trades` as-is (review, don't
   auto-clear). Commit the updated `state.json`.
4. Compute the drawdown ladder tier: live equity ÷ `starting_equity`.
   - ≥95%: 3% risk (regime-dependent, see below)
   - <95%: 2%
   - <90%: 1%
   - <85%: 0.5%
   - ≤80%: **STOP.** Do not scan for entries. Report to the user that
     trading is halted pending explicit reauthorization.

## Universe & liquidity gates (§1–2)
Universe: Nasdaq-listed common stocks ∪ current S&P 500 constituents. No
ETFs, no options. Robinhood's scanner has no exchange filter, so in
practice this screens the broad STOCK universe — note that as a known gap,
don't claim strict Nasdaq-only filtering. Gates (all required, pass/fail):
- Price ≥ $5.00
- Avg $ volume: prefer $250M+, floor $50M+ only if the $250M+ pool is thin
- Acceptable spread, unbroken recent trading, sane/current quote data
- Confirmed tradable on account 831820246 (`get_equity_tradability`)
- No liquidity abnormality — reject anything where market cap and price ×
  volume don't plausibly reconcile (e.g. reported market cap implies a
  wildly different share count than price × volume turnover suggests),
  or where average volume is many multiples of implied float. This has
  caught real bad data before (see journal 2026-08-20 run: SKHY, IPST, UPC
  all excluded this way).

## Ranking rubric (0–100 composite, weighted 0–10 factors)
RS vs SPY 10, RS vs QQQ 8, 20 EMA slope 8, 50 EMA slope 6, distance
above/below 20 EMA (penalize overextension) 8, breakout quality 12, volume
expansion 12, ATR/volatility (favor orderly, penalize chaotic) 6,
momentum/ROC 10, price-action quality 12, verified news catalyst 8.

Grades:
- **A+**: composite ≥85, R:R ≥2.5, top 1% of scan, regime bullish, volume
  confirms, no dangerous event in the holding window. Eligible for
  high-conviction sizing.
- **A**: composite 70–84, R:R ≥2.0. Caps at 75% of calculated risk.
- **B**: composite 55–69. Watchlist only, no trade.
- **C**: composite <55. Discard.

**Trade only A+/A.** A catalyst needs a real sourced event (earnings beat,
guidance, contract, regulatory, confirmed M&A, analyst action) — never "it
moved, must be something."

**Overextension check is not optional.** A name trading 10%+ above its own
20-day EMA is chasing, not a breakout-pullback entry, no matter how good
its other numbers look — this alone disqualified 4 of 7 finalists in the
2026-08-20 run. Also check where today's candle actually closed within its
day's range — a candle that opened near its high and closed near its low
is a same-day fade/reversal, not confirmation, even if the daily %-change
number still looks positive relative to yesterday's close.

## Market regime (§4)
SPY/QQQ/IWM vs their 50 EMA + momentum (ROC). Bullish = ≥2 of 3 above 50
EMA with positive momentum → risk 3%, favor breakouts/continuation.
Neutral = mixed → risk ≤1.5%, max exposure 50%, A+ only. Bearish = ≥2 of 3
below 50 EMA, deteriorating momentum → long-only (no shorting), risk
≤0.5%, max exposure 25%, cash is fine, wait for stabilization.

## Entry sequence (§5) — required in order, don't skip steps
Strong 4H trend → strong 1H trend → break of meaningful resistance →
volume confirms the break → pullback that doesn't destroy the breakout
structure → buyers reappear (reclaim of the level, or higher low with
volume) → entry offers R:R ≥2:1 (prefer 2.5:1+). Never buy the breakout
candle blind. **No-chase rule**: if price is already extended from its
short-term moving average or has already made a large move, wait for a
new base — do not chase, and do not treat "it's up a lot" as a setup.

## Sizing & risk (recompute against live equity every run)
Planned risk/trade: 3% in bullish regime, scaled down by the drawdown
ladder and loss streaks below. Hard per-event cap: never let a single
abnormal event (execution error, gap, halt) cost more than 5% of equity.
Normal max position value: 25% of equity. High-conviction (A+ meeting the
full high-conviction checklist) max: 40% of equity — this changes position
*value* only, never the 3% risk cap; size is always shares = risk$ ÷
stop_distance. Max invested capital: 90% of equity. Max simultaneous
positions: 4, but only fill as many as actually have a qualifying setup —
never pad to fill slots. Check fractional-share eligibility before sizing;
skip the trade if share count rounds to 0 (or <1 on a non-fractional
symbol) rather than inflating risk to force a fill. Higher ATR → wider
stop → smaller size; never raise the risk % to compensate for volatility.

## Daily loss limit & loss streaks (§10, tracked in state.json)
Daily loss limit: 8% of `start_of_day_equity` → once hit, set
`daily_loss_limit_hit_today: true` and stop opening new positions for the
rest of the session. After 3 consecutive losing trades
(`consecutive_losing_trades`): cut risk 50% for the rest of the session.
After 4 consecutive losers: stop opening new positions for the session,
flag for the user to review regime/setup quality before the next session.
Risk never increases after a loss, under any circumstance.

## Stops, targets, trailing (§11)
Every position gets a pre-entry invalidation stop (recent swing low,
breakout level, technical support, or ATR-structural). Never widen a stop,
never average down. Target minimum 2R, prefer 2.5R+; take partials around
2R on strong momentum and trail the remainder. Move the stop to protect
capital at +1.5R, to protect meaningful profit at +2R.

## Concentration, earnings, gaps (§12–14)
Sector exposure normally <60%. Also check 20-day return correlation
between a new candidate and current holdings — treat >0.7 as effectively
the same bet regardless of sector label, and take the strongest of
correlated candidates rather than several. No new normal position within 2
trading days of that symbol's earnings (check via `get_earnings_calendar`
or `get_earnings_results`) — a held position with earnings approaching
gets capped at 10% of equity and lower priority. Gap-ups: don't buy the
gap — wait for the opening range, consolidation, and volume confirmation.
Overnight/24-Hour-Market entries are out of scope for this scheduled
market-open run (see Robinhood's curated ~226-symbol list and §14 rules if
ever extending into overnight monitoring).

## Immutable rules (§17)
Never: use leverage/margin, trade options, average down, martingale,
revenge trade, widen a stop, exceed the drawdown ladder or daily loss
limit, trade illiquid names intentionally, trade on rumors alone, assume a
fill without verifying, concentrate the whole account in one name, force a
trade when none qualifies. **No setup is better than a bad one** — a
"no qualifying setup today" outcome is a normal, correct result, not a
failure to report around.

## What to produce this run
1. Do the Step 0 checks above.
2. Determine regime.
3. Screen the universe, apply liquidity gates, rank and grade candidates
   per the rubric above — with the same depth as a manual run: check 20
   EMA slope and extension, actual regular-session candle structure (not
   just a live/extended-hours snapshot), volume, ATR, and earnings dates
   before finalizing any grade. Robinhood's scanner
   `FILTER_TYPE_PERCENT_CHANGE_FROM_CLOSE` filter takes **fractions, not
   percentages** (0.01 = 1%, not "1") — this bit a previous run.
4. For any A/A+ setup: produce a full order ticket (symbol, grade,
   composite score, entry, stop, target, shares, position value, planned
   risk $, reward:risk, rationale). Since `execution_mode` is
   `signal_only`, do **not** place the order.
5. Append the outcome to `journal.jsonl` (one JSON object per line,
   append-only) — either the ticket(s) produced, or a no-trade summary
   with the actual reasoning (which candidates were considered, why each
   was rejected), following the style of the existing 2026-08-20 entry.
6. Update `state.json`'s session/counter fields as needed and commit +
   push both files back to this repo.
7. Report a concise summary to the user: regime, what was screened, what
   (if anything) qualified, and why.
