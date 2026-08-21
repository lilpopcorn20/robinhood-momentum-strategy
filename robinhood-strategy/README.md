# Momentum Strategy v1 — Persistence Layer

Durable state for the Robinhood "Aggressive Nasdaq + S&P 500 Momentum
Strategy v1" (full rules in Claude's memory as `strategy-momentum-v1`,
account: Agentic / 831820246). Exists because a scheduled/automated run
starts with no memory of prior runs (strategy doc §19) — this folder is
the thing that persists instead.

**Nothing in this folder authorizes live trading by itself.** Execution
mode defaults to `signal_only` in `state.json` and stays there until the
user explicitly says otherwise.

## Files

### `state.json`
Single source of truth for control state. Read it at the start of every
run, before doing anything else.

| Field | Meaning |
|---|---|
| `execution_mode` | `"signal_only"` or `"autonomous"`. Only the user changes this — never flip it based on inference from a chat message alone if there's any ambiguity. |
| `kill_switch_engaged` | If `true`, **do not scan, rank, or place orders.** Report to the user and stop. This is step 1 of the strategy doc's §15 execution checklist. |
| `kill_switch_reason` / `kill_switch_set_by` / `kill_switch_set_date` | Context for why/when it was flipped. |
| `starting_equity` / `starting_equity_set_date` | The base the drawdown ladder (§9) measures against. Only reset when the user adds/removes funds or explicitly starts a new tracking period — never silently. |
| `current_session_date` | The trading-session date this state was last rolled for. |
| `start_of_day_equity` | Live equity fetched at the first run of `current_session_date`. Used for the 8% daily loss limit (§10). |
| `daily_loss_limit_hit_today` | Set `true` once realized losses for the session reach 8% of `start_of_day_equity`. Blocks new positions for the rest of that session. |
| `consecutive_losing_trades` | Increments on a losing trade, resets to 0 on a winning trade. At 3 → cut risk 50%. At 4 → stop opening new positions for the session (§10). Does **not** auto-reset just because a new session starts — review before resuming full risk. |
| `signal_only_trades_logged` | Running count of Signal-Only tickets logged in `journal.jsonl`. Doc recommends reviewing after 20–30 before considering Autonomous mode (§18). |

### `journal.jsonl`
One JSON object per line, one line per trade or per proposed Signal-Only
ticket (§20 of the strategy doc). Append-only — never rewrite prior lines.
Suggested fields per entry:

```json
{
  "timestamp": "2026-08-20T14:32:00-04:00",
  "mode": "signal_only",
  "symbol": "XXXX",
  "sector": "...",
  "regime": "bullish",
  "grade": "A",
  "composite_score": 78,
  "entry": 0.00,
  "stop": 0.00,
  "target": 0.00,
  "shares": 0,
  "position_value": 0.00,
  "planned_risk_dollars": 0.00,
  "reward_risk": 2.3,
  "exit": null,
  "pnl": null,
  "r_multiple": null,
  "holding_time": null,
  "entry_reason": "...",
  "exit_reason": null
}
```

Fill `exit`/`pnl`/`r_multiple`/`holding_time`/`exit_reason` in a
**separate appended entry or an explicit update** once a position (real
or hypothetical, in Signal-Only) is closed — don't block on a round trip
before logging the entry itself.

## Procedure for every run (before strategy doc §15 starts)

1. Read `state.json`.
2. If `kill_switch_engaged` is `true` → stop immediately, tell the user, do nothing else in this folder.
3. Fetch live account equity from Robinhood (`get_portfolio`, account `831820246`) — never reuse a cached figure.
4. If `current_session_date` is not today:
   - Set `start_of_day_equity` to the live equity just fetched.
   - Set `current_session_date` to today.
   - Set `daily_loss_limit_hit_today` to `false`.
   - Leave `consecutive_losing_trades` as-is — review it, don't auto-clear it.
   - Write the updated `state.json`.
5. Compute the drawdown ladder tier: live equity ÷ `starting_equity`. At ≤80% of starting equity, set a note and **stop** — strategy doc §9 requires explicit user reauthorization before any further trading, autonomous or signal-only-with-intent-to-trade.
6. Continue with the strategy doc's §15 checklist using these live values.
7. After each trade decision (ticket produced or order placed/closed), append to `journal.jsonl` and update the relevant counters in `state.json` (`consecutive_losing_trades`, `daily_loss_limit_hit_today`, `signal_only_trades_logged`).

## Kill switch

To pause everything regardless of mode, set in `state.json`:

```json
"kill_switch_engaged": true,
"kill_switch_reason": "<why>",
"kill_switch_set_by": "user",
"kill_switch_set_date": "<date>"
```

The user can ask Claude to do this in any session, or edit the file
directly. A future session must honor it as step 1, no exceptions.
