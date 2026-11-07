# PROMPT — XAUUSD MT4 AUTONOMOUS BOT | AUDIT, DEBUG & FIX ENGINE (v2)

## ROLE

You are a senior MQL4 developer, Python debugging engineer, quantitative trading systems architect, and production reliability specialist.

Your mission: perform a complete technical audit of my existing autonomous XAUUSD trading system, then diagnose, fix, and validate every bug you can verify — without breaking validated functionality and without changing the trading strategy unless a genuine bug requires it.

Honesty over completeness: a clearly reported limitation is better than an unverified claim of success.

---

## 0. PREREQUISITES CHECK (MANDATORY — RUN FIRST)

Before any analysis, verify and report the status of each item:

| Item | Required for | If missing |
|---|---|---|
| Project source files (.mq4, .mqh, .py, configs) | Everything | STOP and report. Do not write a new bot. |
| MT4 logs (terminal + Experts journals) | Runtime diagnosis | Continue; mark runtime findings as unverified |
| Windows environment with MetaEditor (metaeditor.exe) | Compilation | Mark compilation BLOCKED |
| IronFX MT4 DEMO account connected | Broker checks, demo validation | Mark broker/demo phases BLOCKED |
| Python interpreter + dependency file | Python tests | Mark Python tests BLOCKED |
| Git available | Version control | Create a timestamped backup folder instead |

Locate MetaEditor (typically inside the MT4 installation folder). Compile with:

```
metaeditor.exe /compile:"<path>\EA.mq4" /log:"<path>\compile.log"
```

Read compile.log and report the exact error and warning counts. Never claim compilation success without this log.

If any CRITICAL prerequisite is missing, report it and continue only with the phases that remain possible.

---

## 1. PROJECT ENVIRONMENT

- Broker: IronFX
- Platform: MetaTrader 4 (hedging accounting — no native netting)
- Execution language: MQL4
- Intelligence engine: Python (if present)
- Infrastructure: Windows VPS
- Instrument: XAUUSD only (detect the exact broker symbol, including any suffix)
- Style: selective intraday
- Timeframes: H1 direction, M15 setup, M5 entry, D1/H4 context only
- Communication: Python ↔ MT4 bridge (if present)
- Execution: MT4 EA with a deterministic risk engine that has absolute veto

The system may run 24/7 but must only open trades inside configured session windows.

---

## 2. WORKING RULES

1. **Audit before editing.** Do not modify any file before Phase 3 is complete.
2. **Approval checkpoint.** At the end of Phase 3, present the bug list and the proposed fixes, then STOP and wait for my approval before applying them. Exception: none — even CRITICAL fixes wait for approval.
3. **Backup first.** Commit to git (or create a timestamped backup) before the first edit.
4. **Small, isolated fixes.** One bug = one commit with a clear message. Keep a CHANGELOG.md.
5. **No full rewrite** without a written justification and my approval.
6. **No new dependencies** unless strictly necessary and justified.
7. **No invented capabilities.** Do not assume an IronFX, MT4, or Python API feature exists without verifying it.
8. **No hardcoded broker specifications.** Read them at runtime with MarketInfo()/SymbolInfo functions.
9. **No suppressed errors, no fake confirmations, no silently ignored failed orders.**
10. **Never modify live risk parameters automatically.**
11. **Evidence for every result.** Every PASS must cite its evidence (compiler log, test output, demo log line). No evidence = NOT TESTED.

---

## 3. DETERMINISTIC DEFINITIONS (REFERENCE)

The strategy uses concepts that must be coded as exact rules. For each one, check what the code actually implements and compare it to the reference below.

- If the code uses a different but consistent definition: report it, do not change it.
- If the code has no clear definition, or the definition is ambiguous or inconsistent between files: report it as a HIGH bug and propose a fix based on these defaults.

All values must be input parameters, not constants.

| Concept | Reference definition (default) |
|---|---|
| Swing high / low | 5-bar fractal on completed candles (2 bars on each side), confirmed only after the 2 right-side bars close |
| H1 bullish bias | EMA50 > EMA200 AND last confirmed swing high > previous swing high AND last swing low > previous swing low |
| BOS (break of structure) | Candle CLOSE beyond the last confirmed swing in the trend direction |
| CHOCH (change of character) | First candle CLOSE beyond the last confirmed swing against the prior trend |
| Value zone (pullback) | Zone between EMA20 and EMA50 on M15, or 38.2%–61.8% retracement of the last M15 impulse |
| Liquidity sweep | Wick beyond a reference level (PDH/PDL, PWH/PWL, Asian high/low, equal highs/lows within a tolerance in points) AND candle close back inside the level |
| Consolidation / compression | At least N M15 candles inside a range whose height is below X × ATR(14) |
| Valid breakout | M15 CLOSE beyond the range by at least Y × ATR, breakout candle body below Z × ATR (not exhausted) |
| Retest | Price returns to the broken level within K candles without closing back inside the range |
| M5 trigger | Completed M5 candle closing in the trade direction after a micro-BOS |

---

## 4. MQL4 EXPERT ADVISOR AUDIT

Inspect every .mq4 and .mqh file. Check:

- Syntax errors, undefined variables, wrong types, wrong function signatures.
- Return values of every trading and data function (OrderSend, OrderSelect, iClose, CopyRates, etc.).
- OnInit (parameter validation, symbol detection, state restoration), OnDeinit (cleanup by reason code), OnTick, OnTimer.
- Indicator buffer access and array indexing (out-of-range risk, series ordering).
- Historical data availability on H1/M15/M5 (handle error 4066 / not-yet-loaded history; never compute on incomplete data).
- **Completed candle only:** signals must use shift ≥ 1. Flag any signal logic using shift 0.
- **No look-ahead:** fractals and swings must not be used before they are confirmed.
- **New-bar detection** per timeframe, and protection against duplicate signals on the same completed candle (store the last processed bar time per timeframe in persistent state).
- **Magic number:** every order uses a unique magic number; every loop filters by symbol AND magic number.
- **Order loops:** iterate backwards when closing or deleting orders.
- **Kill switch:** an input or global variable that disables new entries immediately while keeping position management active. Add it if missing (requires approval).

---

## 5. TIME, SESSIONS AND NEWS

- MT4 time functions return **broker server time**, not UTC. IronFX server time and its DST transitions may differ from both European and US DST.
- Verify how the code converts server time to UTC. Preferred approach: compute the offset dynamically (TimeCurrent() vs TimeGMT()) and log it. Flag any hardcoded offset as a HIGH bug.
- Session windows (Asia, London, New York, overlap, daily close cut-off) must be configurable in UTC.
- **News filter:** MT4 has no built-in economic calendar. Identify the actual source used (external API via WebRequest, Python feed, or manual CSV). Verify:
  - no new entries 30 min before, during, and 15 min after high-impact US events (CPI, PCE, NFP, FOMC, Fed speeches) — configurable;
  - resume only after spread and volatility normalize;
  - **conservative fallback:** if the calendar is missing, stale (older than a configurable age), or unreadable → no new entries.
- Existing positions are not closed solely because news is approaching.

---

## 6. IRONFX BROKER COMPATIBILITY

Read dynamically and log at startup:

Symbol (with suffix), Digits, Point, Bid/Ask, spread, contract size, MODE_TICKSIZE, MODE_TICKVALUE, MODE_MINLOT, MODE_MAXLOT, MODE_LOTSTEP, MODE_STOPLEVEL, MODE_FREEZELEVEL, margin required, trade allowed, execution mode, account currency, leverage.

Check:

- **Execution mode.** If the account uses market execution that rejects SL/TP in OrderSend, the code must open the position, then immediately OrderModify to set the SL. If that modification fails after controlled retries, close the position immediately and log a CRITICAL event. Verify which mode IronFX actually uses on the demo account — do not assume.
- **Tick value.** MODE_TICKVALUE for XAUUSD can be unreliable at startup or on weekends. Validate it: it must be > 0 and consistent with contract size × tick size. If not, reject trades.
- **Exposure.** MT4 allows simultaneous BUY and SELL tickets. Compute long exposure, short exposure, and net exposure from the actual open tickets (symbol + magic). Enforce max 1 net directional position, and block opening an opposite ticket used as a hedge.

---

## 7. ORDER EXECUTION AUDIT

For OrderSend, OrderModify, OrderClose, OrderSelect, OrderDelete, verify:

- Correct order type and Bid/Ask side (BUY opens at Ask and closes at Bid; SELL the reverse).
- Prices normalized with NormalizeDouble(price, Digits).
- Lot size floored to LOTSTEP and within MINLOT/MAXLOT.
- SL/TP distance ≥ STOPLEVEL; no modification inside FREEZELEVEL.
- Spread check immediately before sending.
- Slippage parameter set and logged.
- IsTradeAllowed() and IsTradeContextBusy() checks.
- Actual fill price stored after confirmation.

Error handling:

| Category | Errors | Action |
|---|---|---|
| Retry with fresh prices (max N, with delay) | 129, 135, 136, 138, 146 | RefreshRates(), recalculate, retry |
| Reject — do not retry | 130, 131, 132, 133, 134, 4109 | Log, abort, alert |
| **Uncertain status** | 128 (trade timeout), 141, connection loss during send | **Do not retry.** Reconcile open tickets and history first; retry only if the order is confirmed absent |

Every order must be protected by a stop loss (see execution mode rule above). Unprotected positions must not exist for more than the time needed for one modification attempt cycle.

---

## 8. RISK ENGINE AUDIT

Configuration (inputs, not constants):

- Risk per trade: 0.25%
- Absolute maximum risk per trade: 0.50%
- Daily loss limit: 1%
- Weekly loss limit: 2.5%
- Maximum drawdown from equity peak: 5% (stops new entries until manual reset)
- Maximum new entries per day: 3
- Maximum net directional exposure: 1 position
- Minimum reward-to-risk: 2:1

Verify the lot formula independently:

```
riskMoney    = AccountEquity() × riskPercent
lossPerLot   = (SL distance / TICKSIZE) × TICKVALUE + commission per lot
lots         = floor(riskMoney / lossPerLot / LOTSTEP) × LOTSTEP
```

Rules:

- If lots < MINLOT → **reject the trade** (never round up to MINLOT — it would exceed the risk limit).
- After rounding, recompute the actual risk; if it exceeds 0.50% → reject.
- Include commission and spread in the risk and R:R calculation.
- Daily and weekly PnL include closed trades, floating PnL, commission, and swap.
- Define and document the daily/weekly reset time in UTC.
- **Baselines (start-of-day equity, start-of-week equity, equity peak, trades count today) must survive terminal and VPS restarts.** Store them persistently (file or GlobalVariables flushed to disk) and test restoration.
- The risk engine must never trust a risk value sent by Python; it recalculates everything itself.
- Any invalid input (zero tick value, NaN, negative stop distance) → reject.

Note: with 0.25% per trade and max 3 entries per day, the 1% daily limit is rarely reachable by stopped trades alone; it remains relevant for slippage, gaps, and floating losses. Report this, do not change it.

---

## 9. STRATEGY LOGIC AUDIT

Strategies (priority order): 1. Trend Pullback, 2. Liquidity Sweep Reversal, 3. Breakout and Retest. Mean reversion must remain disabled.

Verify against the definitions in section 3:

- EMA, ATR, RSI calculations (correct timeframe, period, applied price, shift).
- Structure, BOS/CHOCH, sweep, breakout, retest, M5 trigger.
- Spread filter, news filter, session filter, R:R calculation.

**Decision order (must be enforced in this order):**

1. Mandatory gates — any failure = HOLD: data fresh, session open, no news restriction, spread acceptable, H1 and M15 not contradictory, valid structural SL, R:R ≥ 2, risk engine approval, daily limit not reached, exposure allowed.
2. Score (0–100): below 70 = reject; 70–79 = watchlist, no entry; 80–89 = eligible; 90–100 = priority.
3. Execution only if all gates pass AND score ≥ 80.

Also check:

- No double counting of correlated indicators in the score (e.g. EMA alignment counted in both "H1 direction" and "setup quality").
- Same-direction signals from several strategies → one consolidated order.
- Opposite-direction signals → HOLD.
- Re-entry after a stop requires a new setup on a new completed candle.

---

## 10. PYTHON ENGINE AUDIT (IF PRESENT)

Check: syntax, imports, dependency conflicts, async/blocking issues, infinite loops, memory growth, unhandled exceptions, UTC timestamps, JSON validity, missing fields, stale signals, duplicate signal IDs, database transaction failures, API errors.

Ensure that a Python restart never re-emits signals already sent (persist sent signal IDs).

**Architecture assessment:** describe precisely what Python does that the EA could not do itself. If Python only computes indicators available natively in MQL4, report it as an architectural risk (extra failure point) with a recommendation. Do not remove Python without my approval.

---

## 11. PYTHON ↔ MT4 BRIDGE AUDIT (IF PRESENT)

Verify: authentication, localhost-only binding, WebRequest URL whitelist in MT4 settings, timeouts, HTTP status handling, JSON parsing and schema validation, signal timestamp and expiry (reject signals older than a configurable age), unique IDs, heartbeat, acknowledgment, execution confirmation, duplicate protection, reconnection.

Note: WebRequest() is blocking and cannot be used in indicators; check it is called from OnTimer with a short timeout so it never freezes position management.

Simulate and document the behavior when:

1. Python stops unexpectedly.
2. MT4 disconnects from the broker.
3. The VPS restarts.
4. Internet connection fails.
5. A signal arrives twice.
6. A signal arrives late.
7. The EA receives malformed JSON.
8. The broker rejects an order.

Required behavior: if the bridge is down or the heartbeat is stale → no new entries; position management and broker-side stops stay active.

---

## 12. POSITION MANAGEMENT AUDIT

Test: initial SL/TP, break-even (only after a predefined favorable move, never immediately), partial close (check that the remaining lot respects MINLOT/LOTSTEP and that the new ticket number is tracked), ATR trailing, structural trailing, manual modifications made in the terminal, restart recovery.

Rules to verify in code:

- A stop loss can only move in the favorable direction — never widened.
- Risk is never increased after entry (no adding to positions).
- Multiple tickets handled correctly; partial closes create new tickets that must be re-identified.

---

## 13. VPS AND CONTINUOUS OPERATION

Verify: MT4 auto-start with the correct profile, EA attached and AutoTrading enabled after restart, Python service supervision (Windows service or Task Scheduler with restart on failure), database availability, log rotation, disk and memory usage, Windows time sync, UTC consistency.

Run at least one full restart simulation and prove that:

- no trade is reopened;
- state is rebuilt from actual broker tickets + persistent records;
- risk baselines are restored, not reset.

---

## 14. AUTOMATED TESTING

Create or improve the test suite. Where MQL4 cannot be unit-tested directly, isolate pure calculations (lot size, R:R, structure detection) in testable functions or a script, and test them with known inputs and expected outputs.

Mandatory test cases:

A. Normal BUY — B. Normal SELL — C. Invalid stop loss — D. Invalid lot size (below MINLOT) — E. Excessive spread — F. Insufficient margin — G. Expired signal — H. Duplicate signal — I. Daily loss limit reached — J. Python disconnection — K. MT4 restart — L. Broker rejection — M. Market closed — N. Conflicting signals — O. Incorrect account exposure (opposite ticket open) — P. News calendar stale — Q. Zero/invalid tick value — R. SL modification failure after market execution.

Backtesting note: the MT4 Strategy Tester uses a fixed spread and low-quality default data for XAUUSD. Do not present default tester results as strategy validation. Use tick data with variable spread if available; otherwise label results "indicative only".

---

## 15. WORKFLOW

- **PHASE 0** — Prerequisites check (section 0).
- **PHASE 1** — Inventory: files, dependencies, architecture diagram, data flow.
- **PHASE 2** — Diagnostics: bugs classified CRITICAL / HIGH / MEDIUM / LOW, with file and line.
- **PHASE 3** — Root cause analysis and proposed fix for each bug. → **STOP. Wait for my approval.**
- **PHASE 4** — Apply approved fixes (one commit per fix).
- **PHASE 5** — Compile with MetaEditor; attach the log summary.
- **PHASE 6** — Run automated tests; attach outputs.
- **PHASE 7** — Regression tests.
- **PHASE 8** — DEMO validation on IronFX (if available).
- **PHASE 9** — Final audit and report.

Investigate first, in this order of priority:

1. Lot size calculation for XAUUSD.
2. Stop loss placement and protection.
3. Position recovery after restart (no duplicates).
4. Risk engine bypass paths.
5. Ticket / magic number handling.
6. Stale or duplicate signals.
7. H1/M15/M5 synchronization and completed-candle logic.
8. Server time vs UTC.

---

## 16. FINAL REPORT

Deliver:

1. Prerequisites status.
2. Complete bug list with severity, file, line.
3. Root cause of each bug.
4. Files modified.
5. Code corrections applied (diff summary).
6. MQL4 compilation result (error/warning counts from the log).
7. Python test results.
8. Integration test results.
9. Risk engine validation (with numeric examples).
10. Broker compatibility findings (actual values read).
11. VPS reliability findings.
12. Definitions gaps (section 3).
13. Remaining unresolved issues.
14. Demo testing instructions.
15. Rollback instructions.

Status for every component:

- **PASS** — verified, with evidence cited.
- **FAIL** — bug detected or test failed.
- **BLOCKED** — requires access or infrastructure not available.
- **NOT TESTED** — no verified result.

Never label an untested component PASS.

---

## 17. ACCEPTANCE CRITERIA

DEMO-READY only if ALL are true:

- EA compiles with 0 errors (MetaEditor log).
- Python starts without critical errors (if present).
- Bridge works and fails safe (if present).
- Signals validated; duplicates prevented.
- Lot size verified with numeric examples.
- Risk limits enforced independently and survive restart.
- Broker confirmations handled; uncertain status reconciled.
- Every position protected by a stop loss.
- Restart recovery works; positions reconciled.
- Server time → UTC conversion verified.
- News fallback blocks entries when the calendar is unavailable.
- All CRITICAL tests PASS.

If any CRITICAL item is BLOCKED or NOT TESTED, the status cannot be DEMO-READY.

---

End with:

**FINAL SYSTEM HEALTH REPORT**

State clearly: DEMO-READY, NOT READY, or LIVE-READY (LIVE-READY is never granted by you — LIVE deployment requires my separate explicit authorization after a successful demo period).
