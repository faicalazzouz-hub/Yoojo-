# Audit Prompt — XAUUSD Expert Advisor (MT4, IronFX) — v2

> Run this prompt with Claude Code **on the Windows VPS**, from the project folder,
> so that MetaEditor compilation and demo-account tests can actually execute.
>
> Fields marked `<<FILL>>` are project-specific and must be filled in by the owner
> before the run. If a `<<FILL>>` is left empty, the agent uses the default stated
> next to it and records that it did so.

---

## 0. Role and ground rules

You are auditing an existing MT4 Expert Advisor that trades XAUUSD on an IronFX
account. Your job is to **find and prove defects**, then fix them **only after
approval**.

Hard rules:

1. **Never invent code.** You audit what exists. If a file, function or module is
   missing, report it as missing. Do not write a replacement EA from scratch.
2. **Every PASS needs evidence.** A check is PASS only if you cite one of:
   a compiler log line, a test output, a demo-account log line, or a file path
   with line numbers. No evidence → the result is `UNVERIFIED`, not PASS.
3. **No live account.** All execution tests run on a **demo** account only. If the
   only account available is live, stop and report.
4. **Mandatory pause** after the diagnosis (section 3). No edits before the owner
   approves the bug list.
5. Report in this order of severity: `CRITICAL` (can lose money or open
   unprotected positions) → `HIGH` → `MEDIUM` → `LOW`.

---

## 1. Preflight — check what you actually have

Before any analysis, inventory the environment and print a table:

| Item | How to check | Status |
|---|---|---|
| EA source (`.mq4`) and includes (`.mqh`) | list files in project folder / `MQL4/Experts`, `MQL4/Include` | found / missing |
| MetaEditor compiler (`metaeditor.exe`) | locate the binary; run `metaeditor.exe /compile:<file> /log` | found / missing |
| MT4 terminal + demo account login | terminal running, account type = demo | yes / no |
| Python components (if any) | list `.py` files, `requirements.txt`, how MT4 talks to them | present / absent |
| News calendar source | file, URL, DLL or web request used by the EA | identified / unknown |
| Previous logs (`MQL4/Logs`, `Logs`) | list files and dates | present / absent |

Stop conditions:

- **No EA source code** → stop. Report "nothing to audit". Do not generate an EA.
- **No compiler** → continue with static review only; mark every compile-dependent
  check `UNVERIFIED`.
- **No demo account** → continue without execution tests; mark them `UNVERIFIED`.

---

## 2. Reference definitions (defaults)

Compare the code against these definitions. Where the code differs, **report the
difference** (it may be intentional — say so, but do not silently accept it).
Owner overrides go in the `<<FILL>>` fields.

All values in points/pips assume XAUUSD with 2 decimals (1 pip = 0.10 USD of price
= 10 points). Verify `Digits` and `Point` at runtime; any hardcoded assumption
about them is a bug.

| Concept | Default definition | Owner override |
|---|---|---|
| **Swing high / low** | Fractal of N=2 bars each side on closed bars only (bar index ≥ 1). A swing is confirmed only once the N right-side bars are closed. | `<<FILL>>` |
| **BOS** (break of structure) | A candle **closes** beyond the last confirmed swing in the direction of the current trend. Wick-only breaks do not count. | `<<FILL>>` |
| **CHOCH** (change of character) | First candle **close** beyond the last confirmed swing **against** the current trend. | `<<FILL>>` |
| **Value zone** | Price area between the 50% and 79% retracement of the last impulse leg (swing to swing), or the last opposite candle before the impulse (order block), whichever the code claims to use. | `<<FILL>>` |
| **Liquidity sweep** | Wick exceeds a prior confirmed swing by ≥ 0.1 × ATR(14) plus the current spread, then the **same or next** candle closes back inside. | `<<FILL>>` |
| **Compression** | ≥ 6 consecutive closed bars whose high–low range is ≤ 0.6 × ATR(14) and whose total range is ≤ 1.5 × ATR(14). | `<<FILL>>` |
| **Breakout** | Close beyond the compression box by ≥ 0.2 × ATR(14). | `<<FILL>>` |
| **Retest** | After a breakout, price returns to within 0.2 × ATR(14) of the broken box edge within 10 bars and closes on the breakout side. | `<<FILL>>` |
| **Timeframes** | Signal TF `<<FILL>>` (default M15), bias TF `<<FILL>>` (default H1). | |

Checks for every signal function:

- Uses **closed bars only** (no index 0 values in decision logic) — repainting is
  `HIGH`.
- No look-ahead (e.g. `iHigh(..., 0)` or future-index arithmetic).
- Same result when called twice on the same bar.
- Multi-timeframe calls use `iBarShift` correctly and handle missing history
  (`ERR_HISTORY_WILL_UPDATED` / return 0).

---

## 3. Diagnosis phase (read-only)

### 3.1 Compilation
- Compile with `#property strict`. If it is missing, report it (`MEDIUM`).
- Paste the compiler summary line (`N errors, M warnings`). Every warning is listed
  and classified.

### 3.2 Position sizing — exact formula

The code must be equivalent to:

```
risk_money      = AccountEquity() * risk_pct / 100            // or Balance — state which
tick_value      = MarketInfo(Symbol(), MODE_TICKVALUE)
tick_size       = MarketInfo(Symbol(), MODE_TICKSIZE)
stop_distance   = |entry - stop|                              // in price
commission_lot  = <<FILL>> (round-turn commission per 1.00 lot, account currency)
loss_per_lot    = (stop_distance / tick_size) * tick_value + commission_lot
                  + spread_cost_per_lot (if entry at market)
raw_lots        = risk_money / loss_per_lot
lots            = floor(raw_lots / lot_step) * lot_step        // ALWAYS round DOWN
if lots < min_lot           -> REJECT the trade (never round up to min_lot)
if lots > max_lot           -> cap at max_lot
```

Bugs to look for:
- `tick_value` or `tick_size` ≤ 0 or `EMPTY_VALUE` not rejected → `CRITICAL`.
- Rounding up / `MathMax(lots, min_lot)` → `CRITICAL` (exceeds risk).
- Commission ignored → `HIGH`.
- Using `Point` instead of `MODE_TICKSIZE`, or a hardcoded pip value → `HIGH`.
- Account currency ≠ USD not handled → `MEDIUM`.

### 3.3 Time handling
- Identify the IronFX server time zone and DST rule from `TimeCurrent()` vs
  `TimeGMT()` (or documented broker offset). Print the measured offset.
- All session windows and news times must be converted **server → UTC** with a
  computed offset.
- **Any hardcoded hour offset (e.g. `+2`, `-3`) is a bug** (`HIGH`), because it
  breaks at DST changes.
- `TimeLocal()` (VPS clock) used for trading decisions → `HIGH`.

### 3.4 News filter
- Identify the **actual** calendar source (file path, URL, DLL, indicator).
- Check: what happens if the source is missing, unreachable, or stale?
  Required behaviour: if the calendar is missing or its last update is older than
  `<<FILL>>` (default 24 h), **no new entries are opened**. Existing positions keep
  their protection.
- Check currency/impact filtering (USD high impact at minimum) and the
  before/after blackout windows (`<<FILL>>`, default 30 min / 30 min), in UTC.
- If the filter "fails open" (trades when data is missing) → `CRITICAL`.

### 3.5 Order execution
Classify every MT4 error code the code handles (or fails to handle) into:

| Group | Codes (examples) | Required action |
|---|---|---|
| **Retry** | 4 SERVER_BUSY, 6 NO_CONNECTION, 8 TOO_FREQUENT_REQUESTS, 128 TRADE_TIMEOUT*, 129 INVALID_PRICE, 135 PRICE_CHANGED, 136 OFF_QUOTES, 137 BROKER_BUSY, 138 REQUOTE, 146 TRADE_CONTEXT_BUSY | `RefreshRates()`, bounded retries (≤ 3), back-off |
| **Reject** | 130 INVALID_STOPS, 131 INVALID_TRADE_VOLUME, 133 TRADE_DISABLED, 134 NOT_ENOUGH_MONEY, 148 TOO_MANY_ORDERS, 4109 TRADE_NOT_ALLOWED | do not retry; log and skip |
| **Uncertain status** | 128 TRADE_TIMEOUT, 6 / 4 after send, 141 TOO_MANY_REQUESTS, any unknown code | **before re-sending, scan open orders** (`OrdersTotal`, match magic + symbol + comment / time) to check whether the order was actually filled |

*128 is both: retry is only allowed after the open-orders check confirms nothing was filled.

Stop-loss placement:
- If the broker refuses SL/TP in `OrderSend` (ECN/market execution), the code must
  open with SL=0 then **immediately** `OrderModify` to set the stop.
- If `OrderModify` fails after bounded retries → **close the position** and log it.
  A position left without a stop → `CRITICAL`.
- Check `MODE_STOPLEVEL` and `MODE_FREEZELEVEL` before sending / modifying.
- Magic number used consistently; no handling of other EAs' orders.
- Slippage parameter set and in points appropriate for XAUUSD's digits.

### 3.6 Python components (only if present)
- Explain what Python does that MQL4 cannot. If the answer is "nothing essential",
  recommend removing it: it is an extra point of failure.
- If kept: what happens when the Python process is down, slow, or returns garbage?
  Required: EA does not open new trades, keeps existing protection, logs the fault.

### 3.7 State and restarts
- Behaviour after terminal restart / VPS reboot / reconnect: does the EA rebuild
  its state from open orders, or does it lose track (duplicate entries, orphan
  trailing logic)?
- Global variables / files used for state: written atomically, read safely.

### 3.8 Diagnosis output — then STOP

Produce a numbered bug list:

```
#  | Severity | File:line | Description | Evidence | Proposed fix
```

Then write: **"Waiting for approval. Reply with the bug numbers to fix."**
and stop. Do not modify any file until the owner answers.

---

## 4. Fix phase (only after approval)

- Fix only the approved bug numbers, one commit (or one clearly separated change)
  per bug.
- Recompile after each fix; paste the compiler summary.
- No refactors, renames or "improvements" outside the approved list.

---

## 5. Test phase (demo account)

Each test lists: setup → expected result → evidence (log line / output).

| # | Test | Expected |
|---|---|---|
| T1 | Compile clean | 0 errors; warnings listed |
| T2 | Lot size normal case | Computed lots match the formula in 3.2 within one `lot_step` |
| T3 | Lot below broker minimum | Trade rejected, reason logged, no order sent |
| T4 | Invalid tick value (forced 0 / EMPTY_VALUE) | Trade rejected, no order sent |
| T5 | Stale news calendar (timestamp older than threshold) | No new entries; message logged |
| T6 | Missing news calendar | No new entries; message logged |
| T7 | Inside news blackout window | No new entries |
| T8 | SL refused in OrderSend (simulate or use ECN demo) | SL set by OrderModify right after fill |
| T9 | SL placement fails (simulated `OrderModify` failure) | Position closed, logged |
| T10 | Uncertain-status error after send (simulated) | Open-orders scan runs before any re-send; no duplicate |
| T11 | Terminal restart with an open position | EA resumes management, no duplicate entry |
| T12 | Server→UTC conversion | Logged offset matches broker time; sessions open at the right UTC hour |
| T13 | Signal repaint check | Same signal output when re-evaluated on the same closed bar |

Simulated failures must be done through a test-only input flag (e.g.
`input bool TestForceModifyFail = false;`) that defaults to off; report it so it
can be removed or kept disabled in production.

---

## 6. Backtest

- MT4 Strategy Tester with default history (interpolated ticks, fixed spread)
  is **indicative only**. Label every such result: `INDICATIVE — default MT4 tester`.
- Report modelling quality, spread used, period, and commission setting.
- Do not present tester results as expected live performance.
- If tick data with real variable spread is available, say which source and use it;
  otherwise state it is not available.

---

## 7. Final report format

1. Preflight table (section 1).
2. Definition comparison table: concept → code behaviour → matches default? (section 2).
3. Bug list with severity, evidence, status (`fixed` / `not approved` / `open`).
4. Test table with PASS / FAIL / UNVERIFIED and the evidence for each.
5. Backtest summary, labelled indicative where applicable.
6. Remaining risks and anything you could not verify, with the reason.
