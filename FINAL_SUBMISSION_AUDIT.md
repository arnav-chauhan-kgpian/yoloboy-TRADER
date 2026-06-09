# FINAL SUBMISSION AUDIT — agent.py (frozen)

> Implementation-only audit of the frozen `agent.py`. **No logic, parameter, or design changes are
> proposed or made.** Scope: find implementation bugs. Strategy correctness, parameter choice, and
> design are explicitly out of scope.
>
> Status verified this pass: compiles clean (`py_compile` OK), runs clean on all three preview
> windows (0 errors), **deterministic** (byte-identical across repeated runs), admission **PASSES**
> (gross 0.95×, concentration 29% / 0-day ≥30% streak, worst DD 10.2% < 50%).

## Verdict

**No critical or high-severity implementation bugs found.** The defensive guards, numerical safety,
state persistence, leverage/concentration clamps, and order sequencing are correct. Eight low /
cosmetic observations are listed below; **none affect rule compliance, admission, determinism, or
correctness in the preview/live runners.** The file is submission-safe as frozen.

Severity: **S1** critical · **S2** moderate · **S3** low · **S4** cosmetic.

---

## 1. State persistence

**Correct.** Verified:
- `decide` snapshots all 9 globals (with `dict()` copies of `_pos_high`/`_stop_block`) and restores
  them on any exception (P3). The copies correctly protect against `_run`'s **in-place** mutation of
  `_pos_high` (Phase 6 `del`/assign) and the reassignment of `_stop_block` (Phase 2) — restore yields
  the true pre-call state. Values are floats/ints (immutable), so shallow copies are complete.
- All return paths (Phase 0, Phase 3 `equity<=0`, Phase 10) set `_prev_state`, `_prev_taper_mult`,
  and `_last_seen_date` consistently.
- `prev_cycle_state` (line 605) and `_prev_state` (line 689) both correctly reference last cycle's
  state (captured before Phase 5 overwrites `_state`).
- `_peak_equity` persistence across the continuous live run (and reset per regime via fresh process)
  is the intended, correct behavior.

Findings:
- **F1 [S3] — Counters do not decay on a data-gap tick.** Phase 0 (SPY/QQQ missing/`<MIN_BARS`)
  returns at line 528–541, **before** Phase 2 (line 550). So on a tick where SPY/QQQ are
  uncomputable, `_cooldown` and `_stop_block` are not decremented for that day, while
  `_last_seen_date` is advanced (if a date exists). Effect: a missing-index day "freezes" the
  cooldown/stop-block counters one day. Unreachable at daily cadence with reliable SPY/QQQ (the
  contract guarantees ~220 bars); negligible otherwise. No rule/admission impact.

## 2. Runtime safety

**Correct.** Verified:
- The entire body runs under `try/except Exception → []` in `decide`; no exception can escape and
  forfeit the "runs clean" gate. `BaseException` (KeyboardInterrupt/SystemExit) is intentionally not
  caught.
- All container access is guarded: `.get()` on `market_state`/`last_prices`; iteration over
  `positions`/`LEADER_POOL`; `closes[-1]`/`[-2]` only after `_computable` guarantees `len >= 51`.
- `for bar in spy_bars` (Phase 7) is safe: past Phase 0, `spy_bars` is a non-empty list of dicts
  (proven by the successful `float(bar["close"])` extraction in `_closes_of`); `bar.get("ts")` is
  safe.
- No unbounded loops; per-cycle cost is `O(P·L) ≈ 8–15k` float ops (sub-millisecond, ≫ 5 s headroom).

Findings: none.

## 3. Numerical stability

**Correct.** Verified every division/quotient is guarded:
- `_ret` guards `start <= 0`; `_vol` guards `prev <= 0` and `len(rets) < 2`; `_trend_gap` guards
  `sma50 <= 0`; `_sma` guards `len < n`.
- `sum_raw = n·(n+1)/2` with `n > 0`; `beta_gross` clamp guarded by `beta_gross > 0`; all
  `floor(.../price)` guarded by `price > 0`; `dd` guarded by `_peak_equity > 0` (always true post the
  `equity<=0` check); `breadth` guarded by `n_comp > 0`.
- **Float-equality check `target_w == 0.0` (line 433) is SAFE**, not a bug: weights are only ever
  stored when `> 0.0` (lines 374, 382) and the beta clamp multiplies by a strictly positive scale, so
  `0.0` is exclusively the `.get(ticker, 0.0)` default for absent tickers. The equality correctly
  distinguishes "not in target" from "in target."
- `pstdev` on a constant window returns 0.0 → `vol = 0.0`; downstream comparisons (`< VOL_FULL_MAX`,
  `> BRAKE_VOL10`) behave correctly. No NaN/Inf paths (no `log`, no negative `sqrt`).

Findings: none.

## 4. Leverage calculations

**Correct.** Verified:
- `_beta` defaults to 1.0; `BETA` table covers all 2×/3× names.
- Beta-gross is computed on target weights and clamped to `MAX_BETA_GROSS = 1.45 < 1.5` (lines
  386–389). Leveraged ETFs (QLD/SSO) are added **only** in FULL (lines 333–337, 378).
- Dollar gross ≤ 1.0 by construction: `core_base ≤ 1.0` (`CORE_FULL+SLEEVE_DOLLAR_FULL = 1.0`, or
  `CORE_NEUTRAL = 0.85`) × `taper_mult ≤ 1.0`; sleeve ≤ 0.33; name-caps and the beta clamp only
  reduce. Buys are bounded by spendable cash (no borrow).
- In the admission sample regimes (no QLD/SSO present) beta-gross == dollar-gross ≤ 1.0 — preview
  confirms **peak 0.95×**.

Findings:
- **F2 [S4, informational] — Beta-gross is clamped on _targets_, not on realized holdings.** Between
  rebalances a 2× sleeve name can drift, so realized beta-gross can exceed the 1.45 target by a small
  margin. Bounded well under 1.5 by: the `DRIFT_LIMIT = 0.28` per-name trim (checked every cycle),
  the 3-day cadence, and the fact that appreciation grows the equity denominator too. This is the
  intended target-based control from the blueprint, **not** an implementation defect, and is moot in
  admission (no sleeve). Recorded for completeness.

## 5. Concentration calculations

**Correct.** Verified:
- Target weights capped at `NAME_CAP = 0.26` (lines 373, 381) with no redistribution; drift trim at
  `DRIFT_LIMIT = 0.28` forces a same-cycle rebalance (lines 691–698). Preview confirms **peak 29%,
  longest ≥30% run = 0 days** — inside the "<30% for >5 days" rule with margin.

Findings:
- **F3 [S4, cosmetic] — Drift ratio mixes price sources.** Line 694 computes a position's weight as
  `qty × _exec_price / equity`, where `equity` was built from `_mark_price`. `_exec_price`
  (close→last_prices) and `_mark_price` (last_prices→close) differ in precedence; in the
  preview/live engines `last_prices == latest close`, so the ratio is identical and there is **no
  effect**. Would only matter if a runner supplied `last_prices` diverging from the latest close.

## 6. Order generation

**Correct.** Verified:
- Strict sell-before-buy: forced trailing-stop exits → rebalance sells → buys; the `sold` set
  prevents double-selling a stopped or already-sold ticker.
- `spendable = cash_value + 0.98·proceeds`; buys never exceed it; quantities are integer floors;
  full exits use the exact held quantity (P1); the order cap keeps sells first then weight-ordered
  buys, truncated to 45; a final `quantity > 0` filter drops empties.
- No double-trade: a trimmed name has `deficit < 0` in the buy loop → skipped; a stopped name is in
  `stop_block` → excluded from `weights` → never re-bought; a fully-exited (`target_w==0`) name is
  not in `weights` → never re-bought.

Findings:
- **F4 [S4, cosmetic] — Full-exit sells are emitted without a `market_state` presence check.** In
  `_generate_orders` (line 436), a held non-target ticker is sold even if it's absent from
  `market_state`, whereas the Phase-0 liquidate path guards with `if market_state.get(ticker)`
  (line 532). The engine **silently ignores off-universe orders**, so this is harmless (no error, no
  trade-count inflation that matters), but it can emit a "phantom" sell for a delisted holding that
  can never fill. Cosmetic inconsistency; no rule/admission impact.
- **F5 [S4, informational] — Thrust re-entry changes state immediately but execution waits for the
  rebalance gate.** When the thrust override sets NEUTRAL from CASH, actual buying is governed by the
  unchanged Phase-7 cadence (re-risk is not a `derisk_state`), so deployment can lag the state change
  by up to `REBALANCE_DAYS`. This is the existing, intended cadence behavior (not a defect); noted
  because it is subtle.

## 7. Missing-ticker handling

**Correct.** Verified:
- `_closes_of` returns `None` for absent/malformed series → `_computable` False → the ticker is
  skipped in selection, breadth, sizing, and pricing.
- SPY or QQQ missing/`<MIN_BARS` → Phase-0 liquidate-and-return.
- QLD/SSO missing in FULL → no sleeve; P6 folds the budget into core.
- Position/`last_prices` keys are upper-cased; `market_state` is read with the upper-case universe
  tickers (consistent with the contract's upper-case symbols).

Findings:
- **F6 [S3, known/unfixable] — A held ticker that disappears from `market_state` cannot be exited.**
  `_exec_price` returns `None`, the engine ignores any sell for an off-universe symbol, and the
  position lingers (marked at `avg_cost` in equity). There is no contract-level remedy; negligible in
  a top-liquidity universe. (Same root cause surfaced as F4's cosmetic phantom-order.)

## 8. Edge cases

**Correct.** Verified: empty `market_state` (`return []`), first call (`_last_rebalance_date is None`
→ rebalance), `equity <= 0` (return; effectively unreachable — long-only equity ≥ 0), same-day
re-call (`_last_rebalance_date == current_date` blocks a second rebalance; stops still fire),
first-day `is_new_day`, thrust one-shot (`prev==CASH` → NEUTRAL, won't re-fire next cycle),
`qqq_closes[-2]` length-guarded, `_pos_high` prune/seed, and the thrust branch correctly bypassing
the SMA50-band cash while **never** overriding an active brake (`not brake_fired`).

Findings:
- **F7 [S4, unreachable] — `equity <= 0` early-return is dead/defensive code.** With long-only
  holdings and no borrow, `equity = cash(≥0) + Σ qty(≥0)·price(≥0) ≥ 0`, reaching exactly 0 only in a
  total wipeout that would have tripped the 50% DD gate first. On that (unreachable) path Phase-2
  decay has already run while the state machine has not — a benign inconsistency in code that cannot
  execute.

## 9. Admission risks

**None from the implementation.** All four gates pass with margin and are structurally guaranteed:
- Runs clean — `try/except` envelope + verified 0 errors across all preview windows.
- Leverage — clamp ≤ 1.45×; admission sleeve-free ⇒ ≤ 1.0× (observed 0.95×).
- Concentration — cap 0.26 + drift-trim 0.28 ⇒ peak 29%, 0-day ≥30% streak.
- No blow-up — worst preview DD 10.2% ≪ 50%.
- Trades/day — `MAX_ORDERS = 45`; realistic ≤ ~12.

Findings: none beyond F1–F7 (all S3/S4, none gating).

---

## Consolidated findings table

| ID | Sev | Category | One-line | Impact |
|---|---|---|---|---|
| F1 | S3 | State persistence | cooldown/stop-block don't decay on a data-gap tick | none at daily cadence; unreachable in practice |
| F2 | S4 | Leverage | beta clamp is on targets, not realized holdings | bounded ≪1.5; intended design; moot in admission |
| F3 | S4 | Concentration | drift ratio mixes `_exec_price`/`_mark_price` | none (prices coincide in preview/live) |
| F4 | S4 | Order generation | full-exit sells lack a presence check (vs Phase 0) | none (engine ignores off-universe orders) |
| F5 | S4 | Order generation | thrust state change vs cadence-gated execution | none; intended cadence behavior |
| F6 | S3 | Missing ticker | a delisted holding cannot be sold | negligible; no contract-level fix |
| F7 | S4 | Edge case | `equity<=0` branch is unreachable dead code | none |

## Statement

No implementation bug in any of the nine audited areas rises above **low/cosmetic**. None affects
correctness, determinism, rule compliance, or admission. **The frozen `agent.py` is submission-ready
as-is.** No changes were made by this audit.
