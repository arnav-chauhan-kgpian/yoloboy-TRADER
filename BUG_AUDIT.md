# BUG AUDIT — agent.py ("Spear") vs IMPLEMENTATION_BLUEPRINT.md + competition rules

> Validator-perspective audit of the generated `agent.py`. Findings are ranked by severity.
> **No file changes made.** A patch list follows at the end. Verdict up front, then evidence.

## Verdict

**The implementation is fundamentally sound and PASSES admission cleanly** (verified: peak gross
0.83×, concentration peak 27% / 0-day streak, worst DD 9.3%, runs clean, deterministic). There are
**no hard rule violations, no concentration violations, and no leverage violations.** The findings
below are: one genuine functional bug (dust retention), one genuine whipsaw logic risk, two design
warts inherited from the blueprint, and several low-severity persistence/hygiene items. Two findings
that would change strategy behavior are flagged **DESIGN — needs sign-off** (the user's standing rule
is "do not change parameters/logic" without approval).

Severity legend: **S1** critical (fails a rule / loses money structurally) · **S2** moderate
(behavioral risk, no rule breach) · **S3** low (edge case / cosmetic).

---

## 1. Bugs (functional defects)

### B1 — [S2] Dead-band retains "dust" positions in CASH / de-risk → CASH state is not fully flat
`_generate_orders`, rebalance-sell block (lines ~407-417). For a non-target name (`target_w == 0`),
the full-exit sell is still gated by the dead-band: `if delta < 0 and (-delta)*price >= min_trade`.
So a holding worth **< 3% of equity is never sold**, even when the regime is `CASH`.

- **Why it matters:** in a stress-driven `CASH` transition, several decayed positions (each < 3%)
  can sum to a non-trivial residual long exposure in a state that is supposed to be 100% cash. It
  directly leaks drawdown into the exact regime the strategy exists to avoid — a **rerun-survival**
  liability. Big positions (13-26% at entry) are sold, so the leak is bounded and rare, but it is a
  real correctness gap vs the design intent ("CASH → hold nothing").
- **Faithful to blueprint?** Yes — §11.3 applies the dead-band to all sells. So this is a
  blueprint-level defect surfaced in code. The "tiny stale position not sold" behavior was correct
  for the *old* agent; for Spear's `CASH` state it is wrong.
- **Fix:** bypass the dead-band for full exits of non-target names (always liquidate `target_w == 0`
  holdings). See Patch P1.

### B2 — [S3] Buy-side `spendable` uses the `cash` argument, not `portfolio_state["cash"]`
`_generate_orders` line 420: `spendable = float(cash) + CASH_BUFFER * proceeds`. Blueprint §13 says
the cash source is `portfolio_state["cash"]` (fallback to the `cash` arg). Equity in `_compute_equity`
correctly uses `portfolio_state.get("cash", cash)`, so the two cash sources can theoretically diverge
while sizing buys vs. computing equity.

- **Why it matters:** harmless in practice (the contract guarantees `cash == portfolio_state["cash"]`),
  but it is an unverified precedence divergence — exactly the kind of thing a determinism audit flags.
- **Fix:** thread a single resolved cash value (Patch P2).

### B3 — [S3] Partial global mutation if an exception fires mid-`_run`
`decide` wraps `_run` in `try/except → []`. But `_run` mutates globals incrementally (Phase 2
decrements `_cooldown`/`_stop_block`; Phase 3 updates `_peak_equity`; Phase 5 sets `_state`/`_cooldown`;
Phase 6 mutates `_pos_high`/`_stop_block`). If an exception is raised **after** some of these mutate
but **before** Phase 10, the globals are left in a half-updated state and the next call reads
inconsistent state (e.g., `_state` advanced but `_prev_state`/`_last_seen_date` not).

- **Why it matters:** low probability (defensive guards everywhere), but it can desync the cadence /
  cooldown / peak after a single malformed tick, which is a **determinism/persistence** hazard.
- **Fix:** compute into locals and commit globals in one block at the end, or snapshot/restore on
  exception (Patch P3).

### B4 — [S3] Phase-0 liquidate path leaves `_prev_state` / `_prev_taper_mult` stale
`_run` Phase 0 (data guard) updates `_last_seen_date` but not `_prev_state` or `_prev_taper_mult`.
After a one-tick data gap (SPY/QQQ briefly missing), the next normal cycle compares the new `_state`
against a `_prev_state` from *before* the gap, which can spuriously trigger (or suppress) a
`derisk_state` rebalance.

- **Why it matters:** minor, transient; can cause one unnecessary (or missed) rebalance after a data
  hiccup. Persistence-correctness nit.
- **Fix:** set `_prev_state`/`_prev_taper_mult` consistently on the Phase-0 return (Patch P4).

---

## 2. Logic errors / behavioral risks

### L1 — [S2, DESIGN] No hysteresis gap on the CASH↔NEUTRAL boundary → whipsaw churn
`_run` Phase 5. The CASH trigger is `index < SMA50·(1 − EXIT_BAND)` (= SMA50·0.99). To *leave* CASH,
the trigger must be false, i.e. `index ≥ SMA50·0.99`. **The same single threshold (0.99·SMA50) gates
both directions** — there is no dead-band between exiting to CASH and re-entering NEUTRAL. An index
oscillating around 0.99·SMA50 produces repeated `CASH → full liquidation → NEUTRAL → full re-buy`
cycles, each paying slippage on the whole book.

- **Evidence:** the FULL gate *does* have a gap (enter at SMA50·1.01, leave at SMA50·0.99), but the
  CASH/NEUTRAL gate does not. The 3-day cooldown blocks FULL but **not** NEUTRAL re-entry, so the
  liquidate/re-buy whipsaw still fires in choppy tape. Consistent with the elevated trade counts in
  the selloff (35) and vol-spike (24) preview windows and their negative Calmar.
- **Faithful to blueprint?** Yes — §5 defines the triggers exactly this way. So it's a design gap, not
  a coding error. But it is the single biggest **rerun-survivability** risk (sideways/whipsaw regimes).
- **Fix (DESIGN — needs sign-off):** add a re-entry band or an N-day "index back above SMA50·0.99"
  confirmation before leaving CASH, OR route CASH→NEUTRAL through the cooldown like FULL re-entry.
  See Patch P5. **Do not apply without approval** (changes strategy behavior).

### L2 — [S2, DESIGN] `CORE_FULL (0.67) < CORE_NEUTRAL (0.85)` → FULL *under-deploys* when the sleeve is absent
`_build_targets`. In FULL, core budget is `0.67`, with `0.33` reserved for the QLD/SSO sleeve. When
the sleeve is **not computable** (the admission sample regimes contain no QLD/SSO; a rerun window
might not either), the sleeve budget becomes cash (§8.3, "no redistribution"), so FULL deploys only
`0.67×` while NEUTRAL deploys `0.85×`.

- **Why it matters:** in the *most bullish* state, without the sleeve, the book is **lighter** than in
  the neutral state — backwards. It depresses the numerator (and Calmar) precisely in strong-uptrend
  admission/rerun windows, and nudges toward the `MDD≈0 → Calmar 0` trap. It does **not** cause
  admission failure (under-deployment is safe), only a weaker robustness profile.
- **Faithful to blueprint?** Yes — §8 fixes these budgets and forbids redistribution. Design wart,
  inherited.
- **Fix (DESIGN — needs sign-off):** when no sleeve name is computable in FULL, set
  `core_budget = (CORE_FULL + SLEEVE_DOLLAR_FULL) × taper_mult` (= 1.0×) so the freed budget deploys
  into the 1× core. Patch P6. **Do not apply without approval.**

### L3 — [S3] FULL↔NEUTRAL oscillation resizes the whole core budget (0.67↔0.85) → extra turnover
When the market hovers near the FULL gate (vol≈30%, breadth≈0.5), each FULL↔NEUTRAL flip changes
`core_budget` and triggers a `derisk_state` rebalance that resizes every position. Minor slippage
drag; bounded by the dead-band. Related to L1 but lower severity. No fix required; note for awareness.

---

## 3. Rule violations

**None found.**
- **Long-only:** every order is `buy`/`sell`; sells are bounded by held quantity; no shorting path. ✓
- **Trades/day ≤ 50:** `MAX_ORDERS = 45` hard cap; realistic book ≤ ~7 names ⇒ ≤ ~12 orders/cycle. ✓
- **Min-hold ≥ 60s / runtime ≤ 5s:** daily decisions; per-cycle work is `O(P·L)` ≈ tens of thousands
  of float ops, sub-millisecond. ✓
- **No lookahead:** the agent only reads bars present in `market_state` (today-inclusive in preview,
  prior-close in live); no external data, no future bars. ✓

## 4. Concentration violations

**None found.** Targets capped at `NAME_CAP = 0.26`; `DRIFT_LIMIT = 0.28` forces a same-cycle trim;
the drift check runs **every** cycle. Worst case is ≤ 1 day above 0.28 before the trim fills — far
under the ">5 consecutive days ≥ 30%" rule. Preview confirmed **peak 27%, longest ≥30% run = 0 days.** ✓

## 5. Leverage violations

**None found.**
- Dollar gross ≤ 1.0× by construction (Σ target weights ≤ `CORE_* + SLEEVE_DOLLAR_FULL ≤ 1.0`; buys
  bounded by spendable cash; no borrow).
- Beta-adjusted gross clamped at `MAX_BETA_GROSS = 1.45 < 1.5`; verified 1.32× in the FULL/leverage
  unit test and 0.83× peak in admission preview.
- Leveraged ETFs (QLD/SSO) are reachable **only** in FULL and are absent from the admission sample,
  so admission is leverage-free. **Note (not a violation):** if the *hidden* admission regimes do
  contain QLD/SSO, FULL would engage ~1.33× — still compliant. ✓

## 6. State-persistence mistakes

- **B3** (partial mutation on exception) and **B4** (Phase-0 stale `_prev_*`) above.
- **Intended (not a mistake):** `_peak_equity` persists across the continuous live run (one peak for
  the 30 days) and the taper floor is `0.25` (never 0) — this is the deliberate fix to the
  lockout bug and is correct. Verified re-expandable in the taper unit test.
- **Minor:** `_compute_equity` sums positions in dict-iteration order; deterministic given consistent
  input ordering, but a defensive `sorted()` would harden it against any engine that reorders
  positions between identical runs (determinism guarantee). S3.

## 7. Ticker-availability issues

- Handled well: `.get()` everywhere, `_computable()` gate, graceful degradation to whatever subset is
  present, breadth denominator guarded against zero.
- **T1 — [S3]** A held ticker that **disappears** from `market_state` (e.g., delisted, or dropped from
  a finer-resolution feed) cannot be sold — the engine ignores off-universe orders, and our code
  skips it (`if market_state.get(ticker)` in Phase 0; `price is None` paths elsewhere). We would be
  stuck holding it, marked at `avg_cost` in equity. Very low risk in a top-liquidity universe; no
  action beyond awareness.

## 8. Admission-failure risks

**Effectively none.** All four gates pass with large margins (DD 9.3% vs 50%, gross 0.83× vs 1.5×,
concentration 27%/0-day, clean run). Residual theoretical items:
- If a hidden regime delivers `SPY`/`QQQ` with `< MIN_BARS` (51) bars, the agent liquidates and sits
  in cash → 0 trades. Admission still *passes* (inactivity is not a failure), but the robustness
  profile would be empty. Contract guarantees ~220 bars, so this is remote.
- **L2** lowers the admission robustness *profile* (under-deployment in FULL-without-sleeve) but does
  **not** cause failure.

## 9. Rerun-survivability issues

Ranked by impact:
1. **L1 (whipsaw)** — choppy markets near the 50-SMA churn the book with full liquidate/re-buy cycles;
   slippage drag turns marginal windows into negative Calmar. Highest concern.
2. **B1 (dust in CASH)** — residual long exposure in a state meant to be flat leaks crash drawdown.
3. **L2 (FULL under-deploy without sleeve)** — if a rerun window lacks QLD/SSO, the strongest state
   is the lightest, suppressing Calmar.
4. **Inherent (snapback miss)** — the EXIT_BAND lag + 3-day cooldown keep us out of fast V-recoveries
   (preview vol-spike Calmar −3.50). This is a deliberate denominator-over-numerator trade, not a bug,
   but it caps upside in snapback regimes the rerun will sample.

## 10. Differences from IMPLEMENTATION_BLUEPRINT.md

- **D1 — [intentional]** Sleeve respects `_stop_block` (`_build_targets`, line 345). §8.3 does not
  mention stop-block for the sleeve; this was a documented resolution (#6) for consistency with the
  core. Behavioral, minor, defensible. Flagged for the record.
- **D2** — buy `spendable` cash source (= **B2**), §13 precedence not followed.
- **D3 — [cosmetic]** `assert closes is not None` used for type-narrowing in `_build_targets` and the
  breadth loop. Stripped under `python -O`; harmless because the preceding `_computable()` guarantees
  non-None, but `assert` for control-flow-adjacent narrowing is a code-quality smell. Replace with an
  explicit `if ... continue`/`else` guard (Patch P7).
- **Otherwise faithful:** parameters (§1), universe (§2), state vars (§3 + the documented 9th
  `_prev_taper_mult`), feature math (§4), state machine (§5), taper (§6, floor 0.25), selection/sizing
  (§7-8), order sequencing (§11), and the phase order (§12) all match the blueprint as written.

---

## PATCH LIST

Patches are grouped: **SAFE** (bug/hygiene fixes that do not change strategy parameters or intended
behavior — recommend applying) and **DESIGN** (alter strategy behavior — require explicit sign-off
per the standing "don't change params/logic" rule).

### SAFE fixes (recommend applying)

| ID | Finding | Location | Change |
|---|---|---|---|
| **P1** | B1 dust in CASH | `_generate_orders`, rebalance-sell block (~409-417) | For non-target names (`target_w == 0`), emit a **full-quantity sell regardless of the dead-band**. Keep the dead-band only for *trims* of names that remain in the target. Guarantees `CASH`/de-risk is genuinely flat. |
| **P2** | B2 cash source | `decide`/`_run`/`_generate_orders` (line 420) | Resolve cash once as `port_cash = float(portfolio_state.get("cash", cash))` in `_run` and pass that single value to both `_compute_equity` and `_generate_orders`; use it for `spendable`. |
| **P3** | B3 partial mutation | `_run` / `decide` | Either (a) accumulate all global updates into locals and assign the globals only in Phase 10, or (b) snapshot the 9 globals at the top of `decide` and restore them in the `except` branch before returning `[]`. |
| **P4** | B4 stale `_prev_*` on Phase 0 | `_run` Phase 0 return | Before returning from the liquidate path, set `_prev_state = _state`, `_prev_taper_mult = 1.0` (alongside the existing `_last_seen_date` update). |
| **P7** | D3 `assert` for narrowing | `_build_targets` (~324), breadth loop in `_run` (~last loop of Phase 4) | Replace `assert closes is not None` with an explicit `if closes is None: continue` (functionally identical, robust under `-O`, no code-smell). |
| **P8** | §6 determinism hardening | `_compute_equity` | Iterate `sorted(positions)` when summing equity, to guarantee identical float results even if the engine reorders the positions list. (Optional, S3.) |

### DESIGN changes (DO NOT apply without sign-off — they alter strategy behavior)

| ID | Finding | Proposed change | Rationale / risk |
|---|---|---|---|
| **P5** | L1 CASH↔NEUTRAL whipsaw | Add hysteresis to the CASH exit: require the index to reclaim `SMA50·(1 + ENTER_BAND)` (or hold above `SMA50·0.99` for ≥ 2 trading days) before leaving CASH; or gate CASH→NEUTRAL re-entry behind the existing cooldown. | Cuts whipsaw slippage in sideways tape (biggest rerun risk). **Trade-off:** slower re-entry, more missed early recovery — must be validated on the snapback window so it doesn't worsen vol-spike Calmar. |
| **P6** | L2 FULL under-deploy w/o sleeve | When no sleeve name is computable in FULL, set `core_budget = (CORE_FULL + SLEEVE_DOLLAR_FULL) × taper_mult` so the freed 0.33 deploys into the 1× core. | Removes the "FULL lighter than NEUTRAL" inversion; lifts numerator/Calmar in sleeve-less admission & rerun windows. **Risk:** raises FULL dollar gross to 1.0× in those windows (still 1.0× beta — no leverage breach); changes admission robustness profile, so re-run `preview.py` to confirm caps and DD. |

### Items deliberately NOT patched
- **L3** (FULL↔NEUTRAL resize churn): low severity, bounded by dead-band; would only be addressed as a
  side effect of P5-style hysteresis. No standalone patch.
- **T1** (held delisted ticker unsellable): no robust remedy at the contract level; awareness only.
- **Snapback miss**: intentional denominator-first behavior; not a defect.

---

### Recommended order of operations
1. Apply **P1, P2, P3, P4, P7** (safe), then re-run `python preview.py` + the determinism/leverage/
   stop/taper unit checks — expect identical admission verdict, cleaner `CASH` liquidation.
2. Get sign-off on **P5** and **P6** (they change strategy behavior); if approved, apply and
   re-validate on all three windows plus a ±20% parameter sweep, watching the vol-spike Calmar (P5)
   and the FULL-state caps (P6).

*No file was modified by this audit.*
