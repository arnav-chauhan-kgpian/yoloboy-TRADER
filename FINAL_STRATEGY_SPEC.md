# FINAL STRATEGY SPEC — "Denominator-First Defensive Trend" (Builderr v0)

> Complete, implementation-ready design for the recommended strategy family
> (#9 in [`STRATEGY_LANDSCAPE.md`](STRATEGY_LANDSCAPE.md)). **No code.** Every signal, parameter,
> threshold, and transition is fixed below. A developer can implement `decide()` directly from this
> document without making a single design decision.
>
> Read [`ENVIRONMENT_ANALYSIS.md`](ENVIRONMENT_ANALYSIS.md) for the mechanics this spec depends on.

---

## 0. Objective and hard invariants

**Objective:** maximize 30-day **Calmar = annualized return ÷ max drawdown**, then survive a hidden
out-of-sample rerun. Design intent: **minimize realized max drawdown subject to a small reliable
positive drift.** Win the denominator.

**Hard invariants (must hold every tick — never violated by construction):**
- **Long-only.** No sells beyond held quantity.
- **No leveraged ETFs, ever.** Beta-adjusted gross = dollar gross. Dollar gross ≤ `GROSS_MAX_ON = 1.0×` → never near the 1.5× cap.
- **Per-name weight ≤ 20%** and drift-forced rebalance at 25% → never reaches the 30%/5-day breach.
- **≤ 45 orders per call** → under the 50/day cap.
- **stdlib only**, no network, no LLM, runtime ≪ 5 s.
- **Cadence-robust:** all timing derived from **bar timestamps**, never from a `decide()` call counter.
- **Defensive default:** when data is insufficient or signals are ambiguous, hold cash / defensives — never risk-on.

---

## 1. Universe (fixed ticker sets)

All names below are reliably present in admission (they appear in `sample_regimes.json.gz`) **and**
liquid in the live universe. **Individual single stocks are deliberately excluded from the core** —
ETFs only — to remove idiosyncratic single-name gap risk and keep the denominator smooth.

**Offensive basket `OFFENSE` (13 ETFs), grouped into 4 diversification buckets:**

| Bucket | Tickers |
|---|---|
| `BROAD` | SPY, QQQ |
| `TECH` | XLK, SMH, XLC, XLY |
| `CYCLICAL` | XLF, XLI, XLE |
| `DEFENSIVE` | XLP, XLU, XLV, XLRE |

**Defensive sleeve `DEFENSE` (soft risk-off, ranked subset):** XLP, XLU, XLV
**Hard-stress sleeve `BUNKER`:** XLP, XLU
**Regime reference instruments:** `SPY` and `QQQ` (trend); `QQQ` (vol & brake).

> Any ticker absent from `market_state` on a given tick is skipped (use `.get()`); the strategy
> degrades gracefully to whatever subset is present. If `SPY` or `QQQ` is missing entirely, return
> no orders that tick.

---

## 2. Notation and data hygiene

For a ticker, let its bars (oldest-first) give closes `C = [C₁, …, C_N]`, latest `C_N`.

- **Daily simple return:** `rᵢ = Cᵢ / Cᵢ₋₁ − 1`.
- A series is **valid** only if every close used is `> 0`; if any close ≤ 0 or missing, treat that
  signal as undefined (→ name disqualified / regime defensive).
- **`pstdev`** (population standard deviation) is used everywhere for determinism on short windows.
- All "days" are **trading-day bar counts**, not calendar days.
- **Equity** is computed, not given: `E = cash + Σ qtyᵢ × last_priceᵢ` over held positions
  (use `portfolio_state["last_prices"]`, falling back to `avg_cost` if a price is missing).

**Minimum-history gates:**
- `MIN_BARS_ANY = 50`: if `SPY` or `QQQ` has < 50 bars → **all cash** (return only sells of any held names, else `[]`).
- `MIN_BARS_ON = 126`: RISK_ON is only reachable if both `SPY` and `QQQ` have ≥ 126 bars; otherwise the regime is capped at SOFT.

---

## 3. Signals — mathematical definitions

Each block: **Formula → Parameters → Reasoning → Behavior (Bull / Bear / Sideways / Vol spike).**

### 3.1 Simple moving average (trend)
- **Formula:** `SMA_n(t) = (1/n) · Σ_{i=t−n+1}^{t} Cᵢ`.
- **Parameters:** `SMA_TREND = 100` (index trend gate); `NAME_SMA = 50` (per-name own-trend filter).
- **Reasoning:** the 100-day defines the slow baseline regime (stable, few flips); the 50-day
  confirms an individual ETF is itself trending before we hold it.
- **Behavior:** Bull → price comfortably above SMA (gate open). Bear → price below (gate shut).
  Sideways → price oscillates around SMA (handled by hysteresis, §4). Vol spike → SMA lags badly
  (handled by the fast brake, §3.4 — the SMA is *not* relied on for crash protection).

### 3.2 Blended cross-sectional momentum (selection score)
- **Formula:** for a name, `MOM_fast = C_N / C_{N−68} − 1` (63-day return skipping the last 5 days:
  uses `C_{N−5}` as the end point → `MOM_fast = C_{N−5}/C_{N−68} − 1`), and
  `MOM_slow = C_N / C_{N−126} − 1`. **Blended score:** `S = 0.60·MOM_fast + 0.40·MOM_slow`.
- **Parameters:** `MOM_FAST = 63`, `MOM_FAST_SKIP = 5`, `MOM_SLOW = 126`, weights `0.60 / 0.40`.
- **Reasoning:** ~3-month momentum captures the persistent winner; skipping the last week avoids
  short-term mean-reversion; the 6-month leg confirms durability and damps one-month noise.
- **Behavior:** Bull → positive, ranks risk assets. Bear → negative for most names → few qualify →
  forces SOFT/defensive. Sideways → near zero → marginal qualification, low gross. Vol spike →
  momentum collapses → names disqualify, reinforcing de-risk.

### 3.3 Realized volatility (sizing & vol-target)
- **Formula:** `vol_n(t) = pstdev(r_{t−n+1..t}) · √252`.
- **Parameters:** `VOL_SIZE = 20` (gross sizing + inverse-vol weights), `VOL_BRAKE = 10` (fast brake).
- **Reasoning:** 20-day annualized vol is the standard risk gauge; 10-day reacts fast enough to flag
  a vol explosion.
- **Behavior:** Bull → low vol → larger gross. Bear → elevated vol → smaller gross. Sideways →
  moderate. Vol spike → vol surges → gross auto-cut *before* any binary trigger.

### 3.4 Fast crash brake (multi-day drawdown + vol explosion) — on QQQ
- **Formula:** `R_k = C_N / C_{N−k} − 1`. Brake fires if **any** of:
  `R_3 < BRAKE_R3` **or** `R_5 < BRAKE_R5` **or** `vol_10(QQQ) > BRAKE_VOL10`.
- **Parameters:** `BRAKE_R3 = −0.05` (k=3), `BRAKE_R5 = −0.07` (k=5), `BRAKE_VOL10 = 0.55` (annualized).
- **Reasoning:** the denominator is set in the first 2–3 days of a crash; slow SMAs lag. These
  fast triggers fire ahead of the trend gate and force HARD stress.
- **Behavior:** Bull → never fires. Bear (grind) → usually trend gate handles it first; brake fires
  on the sharp legs. Sideways → rarely fires (single-day chop insufficient). Vol spike → fires
  immediately → collapse to BUNKER.

### 3.5 Index momentum gate (anti-bear-rally) — on QQQ
- **Formula:** `IDX_MOM = C_N(QQQ) / C_{N−126}(QQQ) − 1`. Gate requires `IDX_MOM ≥ INDEX_MOM_MIN`.
- **Parameters:** `INDEX_MOM_MIN = −0.02`.
- **Reasoning:** prevents re-risking into a bear-market bounce when the 6-month index trend is still
  broken — a classic momentum failure mode.
- **Behavior:** Bull → satisfied. Bear → fails → blocks RISK_ON even if price pokes above the SMA.
  Sideways → borderline. Vol spike → fails post-crash, keeping us defensive through the snapback.

---

## 4. Regime detection (the state machine)

Three regimes: **RISK_ON**, **SOFT**, **HARD**. Computed every call from the signals above, plus the
equity-curve guard (§11) and cooldown (§10). Evaluate in this strict priority order:

1. **Data gate.** If `SPY` or `QQQ` missing, or `< MIN_BARS_ANY` bars → **all-cash exit** (sell
   everything held; emit no buys). Skip the rest.
2. **HARD (highest priority).** If the fast brake (§3.4) fires → regime = **HARD**. Set `cooldown =
   COOLDOWN_DAYS`.
3. **Equity-curve guard.** If the drawdown guard is *tripped* (§11) → regime is capped at **SOFT**
   (cannot be RISK_ON), regardless of trend.
4. **Cooldown.** If `cooldown > 0` (and not HARD this tick) → regime capped at **SOFT**; decrement
   cooldown by the number of new bar-days elapsed.
5. **Trend gate with hysteresis** (only if none of the above forced a lower state and ≥ `MIN_BARS_ON`):
   - Let `spy_sma = SMA_100(SPY)`, `qqq_sma = SMA_100(QQQ)`.
   - **Enter/stay RISK_ON** requires **all**:
     `SPY_N > spy_sma·(1 + TREND_BAND)` **and** `QQQ_N > qqq_sma·(1 + TREND_BAND)`
     **and** `IDX_MOM ≥ INDEX_MOM_MIN` **and** brake clear **and** guard clear.
   - **Leave RISK_ON → SOFT** (hysteresis) when **either**:
     `QQQ_N < qqq_sma·(1 − TREND_BAND)` **or** `IDX_MOM < INDEX_MOM_MIN`.
   - **State carry-over:** if currently RISK_ON and neither the strong-on nor the clearly-off
     condition is met (the dead zone inside the band), **stay RISK_ON**. If currently not RISK_ON,
     require the full strong-on condition to enter; otherwise **SOFT**.
6. **Default:** **SOFT** (used whenever trend is ambiguous, history is between 50 and 126 bars, or no
   offensive names qualify).

- **Parameters:** `TREND_BAND = 0.015` (±1.5% hysteresis), `COOLDOWN_DAYS = 3`.
- **Reasoning:** de-risking is fast and easy (any single failing condition drops us down a state);
  re-risking is deliberate and gated (must clear the upper hysteresis band + momentum + brake +
  guard). Asymmetric by design — the safe error (too defensive) is cheap, the fatal error (too
  aggressive) is barred.

**Regime behavior summary:** Bull → RISK_ON, full (vol-scaled) book. Bear → SOFT then HARD on sharp
legs; rarely RISK_ON. Sideways → oscillates RISK_ON↔SOFT inside the band, small book either way.
Vol spike → HARD immediately, then SOFT through cooldown, re-enter only on a clean breakout.

---

## 5. Volatility targeting (gross exposure sizing)

The regime sets *which* book; volatility targeting sets *how large* the RISK_ON book is.

- **Formula (RISK_ON gross):**
  `G_on = clamp( TARGET_VOL / max(vol_20(QQQ), VOL_FLOOR_MKT), GROSS_FLOOR_ON, GROSS_MAX_ON )`.
- **Fixed gross for the other regimes:** `G_soft = 0.30`, `G_hard = 0.15`.
- **Parameters:** `TARGET_VOL = 0.10` (10% annualized), `VOL_FLOOR_MKT = 0.06`,
  `GROSS_FLOOR_ON = 0.20`, `GROSS_MAX_ON = 1.00`.
- **Reasoning:** a **low 10% vol target** is the core denominator lever — far below the house bot's
  14% — because halving portfolio vol roughly halves max drawdown, which (convexly) roughly doubles
  Calmar. Capping gross at 1.0× respects the no-borrow reality; flooring at 0.20× in ON prevents a
  vanishing book (Calmar=0 trap). SOFT/HARD use small fixed gross for simplicity and determinism.
- **Worked points:** QQQ vol 8% → G_on = 1.00 (capped); 12% → 0.83; 20% → 0.50; 30% → 0.33; 50% → 0.20 (floored).
- **Behavior:** Bull (calm) → gross 0.7–1.0×. Bear → SOFT 0.30× / HARD 0.15×. Sideways → gross
  0.4–0.7×. Vol spike → gross auto-cut to floor *then* HARD 0.15×.

---

## 6. Portfolio construction (selection + weighting)

Deterministic, single definition. Produces a target-weight map `W = {ticker: weight}` (weights are
fractions of total equity; their sum = book gross ≤ `G`).

**6.1 Choose the candidate set and gross `G` by regime:**
- **RISK_ON:** candidates = `OFFENSE`; `G = G_on`.
- **SOFT:** candidates = `OFFENSE`; `G = G_soft`. (If fewer than `MIN_NAMES` qualify in step 6.2 → fall back to `DEFENSE` at `G_soft`.)
- **HARD:** candidates = `BUNKER` (XLP, XLU); `G = G_hard`; **skip 6.2 ranking** — hold the available BUNKER names, inverse-vol weighted (§6.3).

**6.2 Qualify and rank (RISK_ON / SOFT):**
- A candidate **qualifies** iff: it is present in `market_state` with a valid series, **and**
  `S > 0` (blended momentum score, §3.2), **and** `C_N > SMA_50(self)` (own uptrend).
- Rank qualifying names by `S` descending; take the top `TOP_N`.
- **Parameters:** `TOP_N = 6`, `MIN_NAMES = 3`.
- If `< MIN_NAMES` qualify in RISK_ON → **downgrade regime to SOFT**. If `< MIN_NAMES` qualify in
  SOFT → use the `DEFENSE` sleeve (whichever of XLP/XLU/XLV are present) at `G_soft`. If even those
  are absent → all cash.

**6.3 Inverse-volatility weighting with caps (deterministic, single pass):**
1. For each selected name `i`: `uᵢ = 1 / max(vol_20(i), VOL_FLOOR_NAME)`.
2. Normalize to gross: `wᵢ = G · uᵢ / Σⱼ uⱼ`.
3. **Name cap:** `wᵢ ← min(wᵢ, NAME_CAP)`.
4. **Bucket cap:** for each bucket `b ∈ {BROAD, TECH, CYCLICAL, DEFENSIVE}`, if
   `Σ_{i∈b} wᵢ > BUCKET_CAP · G`, scale every name in `b` by `(BUCKET_CAP · G) / Σ_{i∈b} wᵢ`.
5. **No redistribution.** Final book gross `Σ wᵢ ≤ G`; the remainder is held as cash (intentionally
   conservative — more cash lowers the denominator).
- **Parameters:** `VOL_FLOOR_NAME = 0.05`, `NAME_CAP = 0.20`, `BUCKET_CAP = 0.45`.
- **Reasoning:** inverse-vol gives risk-parity-lite (calm names get more capital, jumpy names less),
  smoothing the curve; the bucket cap is the explicit fix for "correlated diversification" —
  it forbids the book from becoming a one-factor tech bet even when tech is the momentum winner.
- **Behavior:** Bull → 6 ETFs spread across buckets, tilted to inverse-vol. Bear → defensive sleeve
  only. Sideways → mixed, smaller. Vol spike → BUNKER (XLP/XLU) at 0.15×.

---

## 7. Position sizing → orders (target weights to share orders)

Convert `W` (target weights × equity) into long-only integer-share orders that respect cash.

1. Compute `E` (equity, §2) and the price map `P = {ticker: C_N}` from `market_state`.
2. `target_valueᵢ = Wᵢ · E`; `current_valueᵢ = held_qtyᵢ · Pᵢ`.
3. **Dead-band:** ignore any adjustment with `|target_valueᵢ − current_valueᵢ| < MIN_TRADE_PCT · E`.
4. **Sells first (free cash before buying):**
   - Any held ticker **not** in `W` with `current_value ≥ MIN_TRADE_PCT · E` → sell **all** of it.
   - Any held ticker in `W` that is **overweight** by more than the dead-band → sell
     `floor(excess_value / Pᵢ)` shares (never more than held).
5. **Spendable cash** = `cash + CASH_BUFFER · (expected sell proceeds)`, with `CASH_BUFFER = 0.98`
   (haircut for slippage so we never over-order).
6. **Buys second**, iterating targets in a fixed deterministic order (e.g. sorted by descending
   target weight): for each underweight name, `buy_qty = floor( min(deficit_value, spendable) / Pᵢ )`;
   if `buy_qty > 0`, emit the order and decrement `spendable`.
7. **Cap total orders at `MAX_ORDERS = 45`** (sells prioritized over buys if ever near the cap).
- **Reasoning:** sell-before-buy + the 98% buffer + integer floors guarantee we never request more
  than available cash (the engine would clamp anyway, but this keeps fills predictable and ordering
  deterministic). The dead-band kills turnover and slippage bleed.

---

## 8. Exposure-scaling summary

| Regime | Gross `G` | Book | Per-name cap | Bucket cap |
|---|---|---|---|---|
| RISK_ON | `clamp(0.10/vol₂₀(QQQ), 0.20, 1.00)` | Top-6 OFFENSE, inv-vol | 20% | 45% |
| SOFT | 0.30 (fixed) | Qualifying OFFENSE, else DEFENSE | 20% | 45% |
| HARD | 0.15 (fixed) | BUNKER (XLP, XLU) | 20% | — |
| Cash exit | 0.00 | none | — | — |

Gross is **continuous within RISK_ON** (vol-target) and **steps down** across regimes — the two
mechanisms compound: a vol spike both shrinks `G_on` *and* trips HARD.

---

## 9. Risk-off transitions (de-risking is immediate)

De-risking **bypasses the rebalance cadence** and executes the same tick it is detected. A transition
is "de-risking" when the new regime is lower than the prior regime in the order
`RISK_ON > SOFT > HARD`, **or** when the equity-curve guard trips, **or** when a drift breach occurs.

- **RISK_ON → SOFT:** triggered by trend hysteresis break, `IDX_MOM` failing, `< MIN_NAMES`
  qualifying, guard trip, or cooldown. Action: rebuild book at `G_soft`, immediately.
- **any → HARD:** triggered by the fast brake. Action: liquidate to BUNKER at `G_hard`, immediately;
  arm `cooldown = COOLDOWN_DAYS`.
- **Drift breach (any regime):** if any held name exceeds `DRIFT_LIMIT = 0.25` of equity → immediate
  rebalance to targets.
- **Reasoning:** the denominator is set during stress; waiting for the weekly cadence to de-risk
  would surrender exactly the drawdown we are paid to avoid.

---

## 10. Re-entry logic (re-risking is deliberate)

Re-risking **respects the rebalance cadence** and additional gates — we never chase the first green bar.

- **From HARD:** a `COOLDOWN_DAYS = 3` (trading-day) window begins. While `cooldown > 0`, regime is
  capped at **SOFT**. Cooldown decrements by the count of new bar-days each call.
- **HARD/SOFT → RISK_ON** requires **all** of: cooldown expired; full strong-on trend condition
  (both indices > SMA·(1+band)); `IDX_MOM ≥ INDEX_MOM_MIN`; brake clear; guard clear; **and** the
  rebalance cadence is due (or a drift breach forces it).
- **Reasoning:** asymmetric re-entry avoids the "sell the bottom, buy the bounce, sell the
  re-test" whipsaw that destroys Calmar in V-shaped recoveries. We give up some snapback upside
  (numerator) to protect against re-entering into a failed rally (denominator). Correct trade for
  the objective.
- **Behavior:** Vol spike → out fast, back in slow and only on a confirmed breakout. Bull resumption
  after a dip → re-enter within ~1 cadence once price clears the upper band.

---

## 11. Drawdown protection (portfolio-level equity-curve guard)

An account-level circuit breaker independent of market signals — the mechanical backstop on the
denominator.

- **State:** track running peak equity `peak ← max(peak, E)` each call. Current drawdown
  `dd = E / peak − 1`.
- **Trip:** when `dd < DD_GUARD_TRIP` → set guard *tripped*. While tripped, regime is capped at
  **SOFT** (§4 step 3).
- **Clear:** when `dd > DD_GUARD_CLEAR` (recovered) → clear the guard.
- **Parameters:** `DD_GUARD_TRIP = −0.07`, `DD_GUARD_CLEAR = −0.035`.
- **Reasoning:** even with vol-targeting and brakes, a slow correlated bleed could accumulate; this
  guarantees that once our *own* equity is down 7% from its peak, we shrink risk until we recover —
  bounding realized MDD near the trip level plus the residual book's slippage. Peak resets per
  process, so each admission regime and the live window each get a fresh guard.
- **Behavior:** Bull → never trips. Bear/Vol spike → trips early, holds us in SOFT until recovery.
  Sideways → may trip on a deep oscillation, then clear.

---

## 12. Concentration controls (triple-guarded vs the 30%/5-day rule)

1. **Name cap** `NAME_CAP = 0.20` at construction (§6.3 step 3).
2. **Bucket cap** `BUCKET_CAP = 0.45` (§6.3 step 4) — caps factor concentration, not just single names.
3. **Drift limit** `DRIFT_LIMIT = 0.25` → immediate rebalance if any name drifts past 25% (§9).
- Combined: a position cannot sit at ≥ 30% for 6 consecutive days because (a) targets cap at 20%,
  (b) a drift to 25% forces a same-tick rebalance, and (c) the 5-day cadence rebalances anyway.
  **The breach is structurally unreachable.**

---

## 13. Trade-frequency controls (cadence-robust, low turnover)

- **Rebalance cadence:** `REBALANCE_DAYS = 5` trading days. Compute days elapsed by counting `SPY`
  bar dates strictly after the stored `last_rebalance_date` (timestamp-based — **never** a call
  counter). Rebalance when elapsed ≥ 5, OR a de-risk event (§9), OR a drift breach.
- **Dead-band:** `MIN_TRADE_PCT = 0.02` (skip sub-2%-of-equity adjustments).
- **Order cap:** `MAX_ORDERS = 45`.
- **Reasoning:** weekly rebalancing + dead-bands minimize slippage drag (≥5 bps/round-trip) and
  whipsaw, while immediate de-risk preserves crash protection. Timestamp-based cadence makes the
  bot behave identically at daily, 30-min, or 1-min tick frequencies (it only acts when a new
  trading day appears), neutralizing the cadence-ambiguity trap.

---

## 14. Master parameter table

| Param | Value | Param | Value |
|---|---|---|---|
| `SMA_TREND` | 100 | `TARGET_VOL` | 0.10 |
| `NAME_SMA` | 50 | `VOL_FLOOR_MKT` | 0.06 |
| `TREND_BAND` | 0.015 | `GROSS_FLOOR_ON` | 0.20 |
| `MOM_FAST` | 63 | `GROSS_MAX_ON` | 1.00 |
| `MOM_FAST_SKIP` | 5 | `G_SOFT` | 0.30 |
| `MOM_SLOW` | 126 | `G_HARD` | 0.15 |
| `MOM weights` | 0.60 / 0.40 | `NAME_CAP` | 0.20 |
| `INDEX_MOM_MIN` | −0.02 | `BUCKET_CAP` | 0.45 |
| `VOL_SIZE` | 20 | `VOL_FLOOR_NAME` | 0.05 |
| `VOL_BRAKE` | 10 | `TOP_N` | 6 |
| `BRAKE_R3` | −0.05 | `MIN_NAMES` | 3 |
| `BRAKE_R5` | −0.07 | `DD_GUARD_TRIP` | −0.07 |
| `BRAKE_VOL10` | 0.55 | `DD_GUARD_CLEAR` | −0.035 |
| `COOLDOWN_DAYS` | 3 | `REBALANCE_DAYS` | 5 |
| `DRIFT_LIMIT` | 0.25 | `MIN_TRADE_PCT` | 0.02 |
| `CASH_BUFFER` | 0.98 | `MAX_ORDERS` | 45 |
| `MIN_BARS_ANY` | 50 | `MIN_BARS_ON` | 126 |

All values are **round numbers chosen for ±20% robustness** (per the anti-overfit guidance): the
design must not collapse if any single parameter is shifted 20% — that is an explicit acceptance test (§18.5).

---

## 15. Persistent state (module-level, resets per process/regime)

| Variable | Meaning | Init |
|---|---|---|
| `last_rebalance_date` | SPY bar `ts` at last rebalance | `None` |
| `regime` | last regime (`"on"/"soft"/"hard"`) for hysteresis carry-over | `"soft"` |
| `cooldown` | trading-day countdown after HARD | `0` |
| `peak_equity` | running max equity for the DD guard | `0.0` |
| `guard_tripped` | DD-guard latch | `False` |

> These persist **within** a regime/process and reset between regimes (the engine runs each regime
> fresh). No logic may assume state survives across regimes.

---

## 16. Per-call decision flow (ordered, unambiguous — no code)

1. If `market_state` empty, or `SPY`/`QQQ` missing/`< MIN_BARS_ANY` bars → emit sells for any held
   names (liquidate), update `last_rebalance_date`, return.
2. Compute `E`, update `peak_equity`, compute `dd`, update `guard_tripped` (trip/clear per §11).
3. Compute all index signals on SPY & QQQ: SMA_100, IDX_MOM, vol_20(QQQ), vol_10(QQQ), R_3, R_5.
4. Determine **regime** by the §4 priority ladder (HARD → guard → cooldown → trend hysteresis → SOFT
   default), using the carried-over `regime` for hysteresis. Update `cooldown`.
5. Decide whether to act this tick: **act** if (a) `last_rebalance_date is None`, or (b) days
   elapsed ≥ `REBALANCE_DAYS`, or (c) this is a de-risking transition vs the prior `regime`, or (d) a
   drift breach exists. Otherwise return `[]` (and store the new `regime`).
6. Build candidate set + gross `G` for the regime (§6.1), qualify/rank (§6.2), apply downgrades.
7. Compute target weights `W` via inverse-vol + caps (§6.3).
8. Generate orders from `W` (§7): sells, then cash-bounded buys, dead-banded, capped at 45.
9. If any orders are emitted, set `last_rebalance_date = SPY latest ts`. Store `regime`. Return orders.

---

## 17. Expected behavior matrix (whole strategy)

| | **Bull (sustained up)** | **Bear (sustained down)** | **Sideways / chop** | **Vol spike / crash** |
|---|---|---|---|---|
| Regime | RISK_ON | SOFT, HARD on sharp legs | RISK_ON↔SOFT in band | HARD → cooldown SOFT |
| Gross | 0.7–1.0× | 0.15–0.30× | 0.4–0.7× | →0.15× |
| Book | 6 ETFs, inv-vol, multi-bucket | defensive/BUNKER | small mixed | XLP/XLU |
| Return | small-to-moderate **+** | ~flat to small **−** | ~flat | small **−**, then flat |
| Drawdown | very low (2–4%) | bounded (≤~7% by guard) | low (3–5%) | bounded (≤~10–12%) |
| Calmar effect | strong (low denom, + num) | preserved (small denom) | weak num is the risk | survives — no blow-up |

---

## 18. Failure analysis — how this loses, per regime

For each regime: **how it can lose · maximum expected drawdown · mitigation.**

### 18.1 Bull market
- **How it loses:** under-participates. The 10% vol target + bucket cap keep gross modest and
  forbid piling into the leading (tech) factor, so in a strong tech bull the aggressive,
  concentrated competitors post bigger returns. We can lose the *Round-1* numerator race even while
  our Calmar is healthy.
- **Max expected drawdown:** ~2–4% (shallow pullbacks at <1.0× gross).
- **Mitigation:** the bucket cap still permits a 45% tilt to the winning factor; vol-target lifts
  gross to 1.0× in calm bulls. Accept relative numerator give-up — the rerun makes the aggressive
  bots' bull-only edge non-repeatable, so a strong Calmar with low drawdown still wins on the
  two-stage objective. *(If diagnostics show the live window is a calm bull, `TARGET_VOL` may be
  nudged toward 0.12 — within the robustness band — to capture more upside without surrendering the
  denominator.)*

### 18.2 Bear market (sustained grind down)
- **How it loses:** the 100-day trend gate and 126-day momentum gate lag the top; we carry a small
  book into the first leg before SOFT/HARD engages, and the defensive sleeve (XLP/XLU/XLV) can still
  bleed in a broad de-rating where "defensives" also fall.
- **Max expected drawdown:** ~5–7%, **bounded by the equity-curve guard** (trips at −7% → caps at
  SOFT until recovery). Worst plausible ~8% including slippage.
- **Mitigation:** fast brake catches the sharp legs; index-momentum gate blocks bear-rally
  re-entry; guard mechanically throttles further loss; gross already small. Net: a shallow,
  bounded drawdown and roughly flat return → a modest but positive Calmar, and critically **no
  blow-up** (admission and rerun safe).

### 18.3 Sideways / choppy market
- **How it loses:** this is the **primary Calmar risk** — not drawdown, but a **near-zero numerator**.
  Whipsaw between RISK_ON and SOFT inside the hysteresis band plus slippage can leave return slightly
  negative; if MDD is also tiny, Calmar is small or (if return ≤ 0) negative. There is also the
  `MDD ≈ 0 → Calmar = 0` engine guard if we end up almost flat.
- **Max expected drawdown:** ~3–5%.
- **Mitigation:** the ±1.5% hysteresis band, 5-day cadence, and 2% dead-band are explicitly tuned to
  suppress chop turnover; inverse-vol + bucket spread harvests a small carry from whichever sleeves
  drift up. We deliberately keep a non-trivial risk sleeve (gross floor 0.20× in ON, 0.30× in SOFT)
  so we register a *small positive* return and a *small* drawdown rather than going flat — staying
  off the Calmar=0 trap. Sideways is where we are merely *average*, not where we blow up.

### 18.4 Volatility spike / crash (the true tail)
- **How it loses:** at **daily decision cadence the brake cannot act intraday.** A gap-down (e.g.
  weekend-to-open, or an −8% session) hits whatever book we held *before* the brake confirms on the
  next bar. If we entered the spike at, say, 0.6× gross, the first 1–2 days can cost ~4–7% before we
  liquidate to BUNKER. A failed re-entry into a dead-cat bounce can add a second, smaller wound.
- **Max expected drawdown:** ~8–12% (vs ~30% for buy-and-hold in the COVID sample). This is the
  binding worst case for the whole strategy.
- **Mitigation:** vol-targeting **pre-shrinks** gross as vol rises *before* the spike peaks (a spike
  is usually preceded by rising vol → we're already at 0.3–0.5× entering it); the brake then forces
  HARD within one bar; the guard caps cumulative loss at ~7%; cooldown + deliberate re-entry prevent
  the snapback whipsaw. Crucially, 8–12% ≪ the 50% blow-up line — **admission is never at risk** and
  the rerun's stress window is survived with a recoverable denominator.

### 18.5 Cross-cutting failure modes & mitigations
- **Cadence misread** → neutralized by timestamp-based cadence (acts only on new trading days).
- **Sparse `market_state` (admission ~21 tickers)** → basket is built entirely from reliably-present
  ETFs; `.get()` everywhere; graceful degradation to whatever subset exists.
- **Overfitting / rerun failure** → few parameters, all round numbers; **acceptance test: shift any
  single parameter ±20% and confirm Calmar does not collapse and no regime newly blows up.**
- **Auditability (top-10 human code read)** → transparent rules, no network, no lookahead, no
  hidden data — passes the manual review by construction.

---

## 19. Admission-compliance proof (by construction)

| Admission gate | This strategy | Margin |
|---|---|---|
| Runs clean, no error | defensive defaults, `.get()` guards, try/except around signals | ✓ |
| Gross ≤ 1.5× beta-adj | dollar gross ≤ 1.0×, **no leveraged ETFs** | 0.5× headroom |
| No position ≥ 30% > 5 days | 20% cap + 25% drift force + 5-day cadence | unreachable |
| No > 50% drawdown | worst-case design DD ~8–12% | 4–5× headroom |
| ≤ 50 trades/day | `MAX_ORDERS = 45`, ~6–13 names | ✓ |
| Runtime ≤ 5 s | O(tickers × lookback) stdlib arithmetic | ms-scale |

---

## 20. What is intentionally NOT in this design (and why)

- **No leverage / leveraged ETFs** — they inflate the denominator and decay; forbidden.
- **No individual single stocks in the core** — idiosyncratic gap risk; ETFs only.
- **No mean-reversion / falling-knife buying** — worst tail for Calmar.
- **No ML / fitted models** — the rerun is an out-of-sample kill switch for them; data is too thin.
- **No full covariance optimization** — overfit surface; inverse-vol + bucket caps capture ~90% of
  the diversification benefit with ~10% of the fragility.
- **No external data / VIX feeds** — lookahead/DQ risk in admission; not needed.

---

*Specification complete and frozen. A developer can implement `decide()` directly from §§1–16 with
the parameters in §14 and the state in §15. **No code in this document — implementation is the next,
separate step**, to be validated on `preview.py` plus the ±20% parameter-robustness sweep in §18.5.*
