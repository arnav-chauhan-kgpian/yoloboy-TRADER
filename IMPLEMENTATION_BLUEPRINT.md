# IMPLEMENTATION BLUEPRINT — Candidate B "Spear"

> Complete, deterministic execution specification for the `decide()` agent. **No Python here.** This
> document fixes every value, ordering, and rule so that **two independent developers produce
> functionally identical implementations.** Where the design memo left a range, this blueprint picks
> the single value and states it.
>
> Source of truth for the strategy: [`FINAL_DECISION_MEMO.md`](FINAL_DECISION_MEMO.md) (Candidate B).
> Source of truth for the environment: [`ENVIRONMENT_ANALYSIS.md`](ENVIRONMENT_ANALYSIS.md).

---

## 0. Contract recap (the only I/O)

`decide(market_state, portfolio_state, cash) -> list[order]`, called once per decision cycle.
- `market_state`: `{ticker: [bar, …]}`, daily bars oldest-first, each `{ts, open, high, low, close, volume}`.
- `portfolio_state`: `{cash, positions:[{ticker, quantity, avg_cost}], last_prices:{ticker:price}}`.
- `cash`: float (== `portfolio_state["cash"]`).
- Return: `list` of `{ticker, side:"buy"|"sell", quantity:number}`. `[]` = no action.
- All timing is **date-based** (cadence-robust). All math is **stdlib only**. Runtime ≪ 5 s.

---

## 1. Fixed parameters (constants — never change at runtime)

| Name | Value | Meaning |
|---|---|---|
| `MOM_LONG` | 42 | long momentum lookback (trading days) |
| `MOM_SHORT` | 21 | short momentum lookback |
| `MOM_W_LONG / MOM_W_SHORT / MOM_W_GAP` | 0.50 / 0.30 / 0.20 | momentum-score blend weights |
| `NAME_SMA` | 50 | per-name own-trend SMA |
| `IDX_SMA_FAST` | 20 | index fast trend SMA (SPY, QQQ) |
| `IDX_SMA_SLOW` | 50 | index slow trend SMA (SPY, QQQ) |
| `ENTER_BAND` | 0.01 | hysteresis band to enter FULL (price must exceed slow SMA ×(1+band)) |
| `EXIT_BAND` | 0.01 | hysteresis band to trigger CASH (price below slow SMA ×(1−band)) |
| `VOL_SIZE` | 20 | realized-vol window (regime/sizing) |
| `VOL_BRAKE` | 10 | realized-vol window (fast brake) |
| `VOL_FULL_MAX` | 0.30 | FULL requires QQQ vol_20 (annualized) below this |
| `BRAKE_VOL10` | 0.50 | fast brake: QQQ vol_10 above this |
| `BRAKE_R3` | −0.05 | fast brake: QQQ 3-day return below this |
| `BREADTH_MIN` | 0.50 | FULL requires ≥ this fraction of pool above its 50-SMA |
| `TOP_N_MAX` | 5 | max leaders held in the core |
| `NAME_CAP` | 0.26 | per-ticker target weight cap (< 0.30 rule) |
| `CORE_FULL` | 0.67 | core dollar budget in FULL (rest is the 2× sleeve) |
| `CORE_NEUTRAL` | 0.85 | core dollar budget in NEUTRAL (no sleeve) |
| `SLEEVE_DOLLAR_FULL` | 0.33 | dollar budget for QLD+SSO sleeve in FULL |
| `MAX_BETA_GROSS` | 1.45 | hard clamp on Σ(weight×beta) (cap is 1.5) |
| `DD_HALF` | −0.06 | portfolio drawdown that halves gross |
| `DD_LOCK` | −0.10 | portfolio drawdown that floors gross to `TAPER_LOCK` |
| `TAPER_HALF` | 0.50 | gross multiplier when `DD_HALF ≥ dd > DD_LOCK` |
| `TAPER_LOCK` | 0.25 | gross multiplier when `dd ≤ DD_LOCK` (never 0 — see §6) |
| `TRAIL_STOP` | 0.08 | per-name trailing-stop giveback from in-trade high |
| `STOP_COOLDOWN_DAYS` | 3 | re-entry block (trading days) after a stop-out, per name |
| `REBALANCE_DAYS` | 3 | full re-rank cadence (trading days) |
| `COOLDOWN_DAYS` | 3 | post-CASH cooldown blocking FULL (trading days) |
| `DRIFT_LIMIT` | 0.28 | held weight above this forces a rebalance/trim |
| `MIN_TRADE_PCT` | 0.03 | dead-band: skip trades smaller than this × equity |
| `CASH_BUFFER` | 0.98 | fraction of expected sell proceeds counted as spendable |
| `MAX_ORDERS` | 45 | hard cap on orders returned per cycle |
| `MIN_BARS` | 51 | min bars required to compute any name's full feature set |
| `SLIP_EQUITY / SLIP_LEV` | 0.0005 / 0.0010 | slippage estimates (proceeds haircut only; informational) |

All values are deliberate round numbers chosen for ±20% robustness. **No value here is computed or
overridden at runtime.**

---

## 2. Universe (exact, fixed lists)

Filtered at runtime to whatever is present in `market_state` with ≥ `MIN_BARS` bars. Anything absent
is silently skipped (graceful degradation — admission delivers ~21 tickers, live ~1000).

- `INDEX_REF` = `["SPY", "QQQ"]` — regime reference (both required; if either missing → liquidate, §13).
- `LEADER_STOCKS` = `[NVDA, MSFT, AAPL, META, AMZN, GOOGL, AVGO, AMD, MU, MRVL, NFLX, TSLA, PLTR, ORCL, CRM, JPM, V, MA, COST, LLY]`.
- `LEADER_ETFS` = `[QQQ, SPY, SMH, XLK, XLC, XLY, XLF, XLI, XLE, XLV, XLP, XLU, XLRE, DIA, IWM, SOXX]`.
- `LEADER_POOL` = `LEADER_STOCKS ∪ LEADER_ETFS` (ranking candidates).
- `SLEEVE` = `["QLD", "SSO"]` (2× ETFs; used only in FULL; absent in admission → sleeve simply not added → beta-gross ≤ 1.0 there).
- `BETA` table (for the gross clamp): `{QLD:2, SSO:2, TQQQ:3, SOXL:3, UPRO:3, SPXL:3, TNA:3, FAS:3, TECL:3, LABU:3, CURE:3, DRN:3, UDOW:3, NAIL:3, DDM:2, ROM:2, UWM:2, AGQ:2}`; everything else = `1`. (We only ever *buy* 1× names + QLD/SSO; the table is a safety guard.)

> **Note:** the agent never buys 3× ETFs and never buys any leveraged name in NEUTRAL/CASH. Leverage
> (QLD/SSO) is reachable **only** in FULL. This makes admission automatically leverage-free.

---

## 3. State variables (persist across calls within a process; reset between regimes by the engine)

| # | Name | Type | Initialization | Update rule | Persistence |
|---|---|---|---|---|---|
| 1 | `g_state` | enum {`FULL`,`NEUTRAL`,`CASH`} | `NEUTRAL` | set in Phase 5 each cycle | required |
| 2 | `g_cooldown` | int ≥ 0 | `0` | set to `COOLDOWN_DAYS` on entry to CASH; decremented by 1 on each **new trading day** (Phase 2) | required |
| 3 | `g_peak_equity` | float | `0.0` | `max(g_peak_equity, equity)` each cycle (Phase 3) | required |
| 4 | `g_pos_high` | dict{ticker→float} | `{}` | per held name: `max(prev, latest_close)`; created on first hold; deleted when position fully sold | required |
| 5 | `g_stop_block` | dict{ticker→int} | `{}` | set to `STOP_COOLDOWN_DAYS` when a name is stopped out; decremented per new trading day; entry removed at 0 | required |
| 6 | `g_last_rebalance_date` | str(date) or None | `None` | set to `current_date` whenever a full rebalance emits ≥1 order | required |
| 7 | `g_last_seen_date` | str(date) or None | `None` | set to `current_date` at end of every cycle; used to detect a new trading day | required |
| 8 | `g_prev_state` | enum | `NEUTRAL` | snapshot of `g_state` from the **prior** cycle, read in Phase 7 to detect de-risk transitions; updated at end of cycle | required |

> **Persistence semantics:** these are module-level globals. They survive across `decide()` calls
> within one process. The engine runs each admission regime in a **fresh process** (globals reset to
> the initializers above); the live run is **one continuous process** (globals persist the whole
> 30 days). No logic may assume cross-regime carryover. (This is why the drawdown taper in §6 never
> goes fully to cash — see the re-expandability argument there.)

---

## 4. Feature helpers (mathematical definitions — pure functions, no state)

Let `C = [C₁ … C_N]` be a ticker's closes (oldest-first), latest `C_N`. All return a real number or
`UNDEF` (insufficient/invalid data). A series is invalid if any used close ≤ 0 or missing.

- `sma(C, n)` = mean(`C[N−n+1 … N]`); `UNDEF` if `N < n`.
- `ret(C, k)` = `C_N / C_{N−k} − 1`; `UNDEF` if `N < k+1` or `C_{N−k} ≤ 0`.
- `vol(C, n)` = `pstdev(r) × √252`, where `r = [C_i/C_{i−1} − 1 for i in N−n+1 … N]` (n returns);
  `UNDEF` if `N < n+1` or any `C_{i−1} ≤ 0`. Use **population** stdev (divides by n) for determinism.
- `trend_gap(C)` = `C_N / sma(C,NAME_SMA) − 1`; `UNDEF` if `sma` is `UNDEF`.
- `momentum_score(C)` = `MOM_W_LONG·ret(C,MOM_LONG) + MOM_W_SHORT·ret(C,MOM_SHORT) + MOM_W_GAP·trend_gap(C)`;
  `UNDEF` if any component is `UNDEF`.
- `date_of(ts)` = first 10 chars of the bar timestamp string (`"YYYY-MM-DD"`). ISO dates compare
  lexicographically, so string comparison == chronological comparison.

A ticker is **computable** iff present in `market_state`, has ≥ `MIN_BARS` valid bars, and its latest
close `> 0`.

---

## 5. State machine (text diagram)

Three market states. The drawdown taper (§6) and per-name trailing stops (§8) are **orthogonal
overlays** that further reduce/cancel positions but do **not** change the state name.

```
                ┌─────────────────────────────────────────────────────────────┐
                │  Evaluated FRESH every cycle, in this strict priority order:  │
                └─────────────────────────────────────────────────────────────┘

   (A) CASH-TRIGGER?  =  brake_fired
                         OR SPY_close < SMA50(SPY)·(1−EXIT_BAND)
                         OR QQQ_close < SMA50(QQQ)·(1−EXIT_BAND)
        ── yes ──►  g_state = CASH ;  g_cooldown = COOLDOWN_DAYS
        │
        └─ no ─►  (B) g_cooldown > 0 ?
                       ── yes ──►  g_state = NEUTRAL          (FULL is blocked during cooldown)
                       │
                       └─ no ─►  (C) FULL-CONDITIONS?  (ALL must hold)
                                       SPY_close > SMA20(SPY)
                                       AND QQQ_close > SMA20(QQQ)
                                       AND SPY_close > SMA50(SPY)·(1+ENTER_BAND)
                                       AND QQQ_close > SMA50(QQQ)·(1+ENTER_BAND)
                                       AND breadth ≥ BREADTH_MIN
                                       AND vol_20(QQQ) < VOL_FULL_MAX
                                  ── yes ──►  g_state = FULL
                                  └─ no ───►  g_state = NEUTRAL

   where:
     brake_fired = (ret(QQQ,3) < BRAKE_R3) OR (vol(QQQ,VOL_BRAKE) > BRAKE_VOL10)
     breadth     = (# pool names computable AND close > SMA50(self)) / (# pool names computable)

   STATE → BOOK (before taper & stops):
     CASH    →  hold nothing  (target = {} ; sell everything)
     NEUTRAL →  core leaders only, core budget = CORE_NEUTRAL, NO sleeve
     FULL    →  core leaders, core budget = CORE_FULL, PLUS QLD/SSO sleeve = SLEEVE_DOLLAR_FULL

   ORTHOGONAL OVERLAYS:
     • drawdown taper  : multiplies ALL target weights by taper_mult ∈ {1.0, 0.5, 0.25}  (§6)
     • trailing stop   : cancels an individual held name + blocks its re-entry STOP_COOLDOWN_DAYS (§8)

   COOLDOWN (re-entry) path:   CASH ──(cooldown counts down on new trading days)──► NEUTRAL ──(FULL-CONDITIONS)──► FULL
```

**Hysteresis is encoded three ways:** (1) ENTER_BAND vs EXIT_BAND around the 50-SMA (enter high, exit
low); (2) cooldown forces a stop in NEUTRAL before returning to FULL; (3) the fast SMA(20) gate on
top of the slow SMA(50) gate for FULL.

---

## 6. Drawdown taper (orthogonal gross multiplier — re-expandable, never latched)

- `dd = equity / g_peak_equity − 1` (with `g_peak_equity` updated to include the current equity first).
- `taper_mult` is a **pure function of the current `dd`** (recomputed every cycle, no memory):
  - `dd > DD_HALF (−0.06)` → `1.00`
  - `DD_LOCK (−0.10) < dd ≤ DD_HALF` → `0.50`
  - `dd ≤ DD_LOCK` → `0.25`
- Applied by multiplying **every** target weight (core + sleeve) by `taper_mult`.
- **Why the floor is 0.25, not 0:** the taper keys off our **own equity**, which cannot recover while
  fully in cash (equity frozen → `dd` frozen → permanent lockout). Keeping a 0.25 floor lets the book
  participate in a rebound so `dd` can improve and the taper re-expand. *Market* de-risking to full
  cash is a separate mechanism (§5 state = CASH) that keys off **index prices**, which remain
  observable in cash, so re-entry is possible there. **Do not merge these two; do not let the taper
  reach 0.**

---

## 7. Momentum ranking & leader selection (Phase 8a)

1. Build `candidates` = every `LEADER_POOL` ticker that is **computable** (§4) **and** `g_stop_block`
   does not contain it (stop-blocked names are excluded from (re)entry).
2. For each candidate compute `s = momentum_score(C)`. **Qualify** iff `s > 0` **and**
   `C_N > sma(C, NAME_SMA)` (own uptrend).
3. Sort qualifiers by `s` **descending**; tie-break by **ticker string ascending** (deterministic).
4. `selected` = first `min(TOP_N_MAX, len(qualifiers))`. `N = len(selected)`. If `N == 0`, the core
   book is empty (cash core) — not an error.

---

## 8. Position sizing & exposure scaling (Phase 8b → target weights `W`)

Construct `W = {ticker → target weight}` (fraction of equity). Procedure depends on `g_state`:

**If `g_state == CASH`:** `W = {}` (sell everything). Skip the rest of §8.

**Else (NEUTRAL or FULL):**
1. `core_budget = (CORE_FULL if FULL else CORE_NEUTRAL) × taper_mult`.
2. **Conviction (rank-linear) core weights** over `selected` (rank 1 = highest score):
   `raw_i = (N − rank_i + 1)`, `sum_raw = N·(N+1)/2`,
   `w_i = core_budget · raw_i / sum_raw`, then `w_i ← min(w_i, NAME_CAP)`. **No redistribution** of
   capped excess (it becomes cash). Add each `w_i` to `W`.
3. **Sleeve (FULL only):** for each `s ∈ SLEEVE` that is computable:
   `w_s = (SLEEVE_DOLLAR_FULL × taper_mult) / (count of computable sleeve names)`, then
   `w_s ← min(w_s, NAME_CAP)`. Add to `W`. (If neither QLD nor SSO is computable, no sleeve — core only.)
4. **Beta-gross clamp:** `beta_gross = Σ_{t∈W} W_t · BETA[t]`. If `beta_gross > MAX_BETA_GROSS`,
   multiply **every** `W_t` by `MAX_BETA_GROSS / beta_gross`. (By construction this rarely binds:
   FULL ≈ 1.33×.)

> Resulting invariants: every `W_t ≤ NAME_CAP (0.26) < 0.30`; `Σ W_t (dollar) ≤ 1.0`;
> `Σ W_t·BETA_t ≤ MAX_BETA_GROSS (1.45) < 1.5`. Concentration & leverage are unbreachable by design.

---

## 9. Trailing-stop updates (Phase 6 — runs EVERY cycle, before any rebalance gating)

For each ticker currently held (`quantity > 0`):
1. `latest = C_N(ticker)` if computable, else `last_prices[ticker]`, else skip (cannot evaluate).
2. `g_pos_high[ticker] = max(g_pos_high.get(ticker, latest), latest)`.
3. **Stop-out** if `latest < g_pos_high[ticker] · (1 − TRAIL_STOP)`:
   - Mark ticker for **forced full sell** this cycle (added to the sell list in Phase 11 regardless of
     rebalance gating).
   - Set `g_stop_block[ticker] = STOP_COOLDOWN_DAYS`.
   - Remove `ticker` from `g_pos_high` (position is being closed).
4. For a position opened this cycle (a new buy), initialize `g_pos_high[ticker] = latest` after fills
   are conceptually applied — in practice, since fills happen next open, initialize on the **next**
   cycle's Phase 6 from the now-held position. (Net effect: `g_pos_high` is seeded from the first
   close at which the name appears in `positions`.)

Forced stop-out sells are **immediate** (they bypass the rebalance cadence). Freed cash is **not**
redeployed until the next full rebalance.

---

## 10. Re-entry cooldown handling (Phase 2 + Phase 5)

- On every cycle, compute `current_date = date_of(SPY latest bar)`.
- `is_new_day = (current_date != g_last_seen_date)`.
- **If `is_new_day`:** decrement `g_cooldown` by 1 (floor 0); decrement every `g_stop_block` value by
  1 and delete entries that reach 0.
- Entry to CASH (Phase 5) sets `g_cooldown = COOLDOWN_DAYS`.
- While `g_cooldown > 0`, FULL is unreachable (state pinned at ≤ NEUTRAL).
- Decrementing keys off **date changes only**, so cooldown means the same thing at daily, 30-min, or
  1-min tick cadence.

---

## 11. Order generation & sell-before-buy sequencing (Phase 11)

Inputs: target `W`, current positions (aggregated, §14), `equity`, price map `P` (latest computable
close per ticker), `forced_stop_sells` (§9), `cash`.

1. **Forced stop sells** (always, even on non-rebalance cycles): for each stopped ticker held, emit
   `sell` of its full quantity.
2. **Decide whether to run a full rebalance this cycle** (Phase 7 gate, §12). If **not** rebalancing,
   skip to step 7 with only the forced stop sells.
3. **Rebalance sells:** for each held ticker `t`:
   - `target_shares = floor(W.get(t,0) · equity / P[t])` (0 if `t∉W` or `P[t]` missing).
   - `delta = target_shares − held_qty`.
   - If `delta < 0` **and** `|delta|·P[t] ≥ MIN_TRADE_PCT·equity` → emit `sell` of `min(|delta|, held_qty)`.
   - (Tickers not in `W` ⇒ target 0 ⇒ full sell, subject to the dead-band.)
   - Drift/trim is automatic here: an overweight held name has `target_shares < held` → sold down to ≤ `NAME_CAP`.
4. **Compute spendable cash:** `spendable = cash + CASH_BUFFER · Σ(all sell_qty · P[t])` (stop sells +
   rebalance sells).
5. **Rebalance buys**, iterating `W` in **descending target-weight order** (tie-break ticker
   ascending):
   - `target_shares = floor(W[t] · equity / P[t])`; `deficit = target_shares − held_qty`.
   - If `deficit > 0` **and** `deficit·P[t] ≥ MIN_TRADE_PCT·equity`:
     `buy_shares = floor( min(deficit·P[t], spendable) / P[t] )`; if `buy_shares > 0`, emit `buy`,
     `spendable −= buy_shares·P[t]`.
6. **Mark rebalanced:** if this was a full rebalance and ≥1 order was emitted, set
   `g_last_rebalance_date = current_date`.
7. **Order cap:** if total orders > `MAX_ORDERS`, keep **all sells first**, then buys in descending
   target weight, truncating to `MAX_ORDERS`.
8. Return the order list (possibly empty).

> **Long-only / no-borrow:** buys are bounded by `spendable`; quantities are integer floors; sells
> never exceed held quantity. The engine would clamp anyway, but these rules keep fills deterministic.

---

## 12. Order of execution (the master cycle — exact phase sequence)

Every `decide()` call runs these phases **in this order**:

1. **Phase 0 — Guard:** if `market_state` empty, or `SPY`/`QQQ` absent or not computable (< `MIN_BARS`)
   → run the **liquidate path** (emit full sells of every held, computable ticker), update
   `g_last_seen_date`, persist, return.
2. **Phase 1 — Ingest:** aggregate `positions` (§14), build `held_qty`, `last_prices`, price map `P`.
3. **Phase 2 — Calendar:** `current_date`; `is_new_day`; if new day, decrement `g_cooldown` and
   `g_stop_block` (§10).
4. **Phase 3 — Equity & taper:** compute `equity` (§13); `g_peak_equity = max(g_peak_equity, equity)`;
   `dd`; `taper_mult` (§6). If `equity ≤ 0` → persist, return `[]`.
5. **Phase 4 — Index features:** `SMA20/SMA50` for SPY & QQQ; `vol_20(QQQ)`; `ret(QQQ,3)`;
   `vol_10(QQQ)`; `breadth` over `LEADER_POOL`.
6. **Phase 5 — State machine:** compute `brake_fired`, cash-trigger, full-conditions; set `g_state`;
   set `g_cooldown` on CASH entry (§5).
7. **Phase 6 — Trailing stops:** update `g_pos_high`; collect `forced_stop_sells` and set
   `g_stop_block` (§9).
8. **Phase 7 — Rebalance gate:** `do_rebalance = TRUE` if **any** of:
   - `g_last_rebalance_date is None`, OR
   - `days_since_rebalance ≥ REBALANCE_DAYS` (count SPY bar-dates strictly after
     `g_last_rebalance_date`), OR
   - `g_state` is **lower** than `g_prev_state` in the order `FULL>NEUTRAL>CASH` (de-risk transition), OR
   - `taper_mult` **decreased** vs the value implied last cycle (treat any `taper_mult < 1.0` with a
     new lower band as a de-risk — simplest deterministic rule: rebalance whenever `taper_mult < 1.0`
     and we are not already at target), OR
   - any held name's current weight `> DRIFT_LIMIT`.
   **Same-day guard:** if `g_last_rebalance_date == current_date`, set `do_rebalance = FALSE` (never
   full-rebalance twice on one trading date; stop sells still fire).
9. **Phase 8 — Targets:** if `do_rebalance`, build `W` (§7 selection → §8 sizing). Else `W` unused.
10. **Phase 9 — Orders:** generate per §11 (forced stop sells always; rebalance sells+buys only if
    `do_rebalance`); apply `MAX_ORDERS` cap.
11. **Phase 10 — Persist:** set `g_prev_state = g_state`; `g_last_seen_date = current_date`; (the
    `g_pos_high` for newly-bought names is seeded next cycle from `positions`). Return orders.

---

## 13. Cash & equity handling (precise)

- `equity = cash + Σ_{held t} qty_t · price_t`, where `price_t = last_prices[t]` if present and `>0`,
  else latest computable close `C_N(t)`, else `avg_cost_t`, else `0`.
- `cash` argument and `portfolio_state["cash"]` are identical; use `portfolio_state["cash"]` (fallback
  to the `cash` arg if the key is missing/invalid).
- Spendable for buys = `cash + CASH_BUFFER · expected_sell_proceeds` (§11.4). Never exceed it.
- All order quantities are **non-negative integers** (`floor`). Drop any order with quantity 0.

---

## 14. Missing-ticker & data hygiene rules

| Situation | Rule |
|---|---|
| `SPY` or `QQQ` absent / `< MIN_BARS` / invalid | **Liquidate** all held computable names, return (Phase 0). Cannot assess regime. |
| A `LEADER_POOL` name absent / `< MIN_BARS` / close ≤ 0 | Not computable → excluded from candidates & breadth denominator. |
| `QLD`/`SSO` absent in FULL | Sleeve simply omitted; core-only book (lower beta). Not an error. |
| Held ticker absent from `market_state` | Cannot price or emit a valid order for it (engine ignores off-universe). Exclude from equity priced at `avg_cost`; cannot sell until it reappears. Log nothing; continue. |
| Duplicate position entries for one ticker | **Aggregate** quantities; weighted-average `avg_cost`. |
| `last_prices` missing for a held name | Fall back to latest close, then `avg_cost`, then 0. |
| Bar with non-positive/missing close in the window | Series invalid for that feature → feature `UNDEF` → name not computable. |
| `decide()` called twice on same `current_date` (finer cadence) | `is_new_day=FALSE` (no cooldown/stop decrement); `do_rebalance=FALSE` via same-day guard; only trailing stops can act. |

**Every dictionary/list access must be defensive** (`.get`, length checks, try/guard around float
casts). A malformed bar must never raise; it degrades to `UNDEF`/skip.

---

## 15. Error handling

- The **entire body** of `decide()` is wrapped so that **any** unexpected exception results in
  returning `[]` (do nothing this cycle) rather than propagating — a raised exception forfeits the
  tick and risks the "runs clean" admission gate. (Internally, prefer explicit guards over relying on
  the outer catch.)
- Numeric guards: never divide by zero (`prev_close > 0`, `equity > 0`, `sum_raw > 0`, `beta_gross > 0`
  before dividing).
- State is updated **only** through the rules in §3; never partially mutate state before an early
  return without also updating `g_last_seen_date`/`g_prev_state` consistently (Phase 10 must run on
  every non-Phase-0 return path; Phase 0 updates `g_last_seen_date` before returning).

---

## 16. Edge-case handling rules (consolidated)

1. **First-ever call:** `g_last_rebalance_date is None` ⇒ `do_rebalance = TRUE`. Likely state NEUTRAL
   (cooldown 0, FULL needs full conditions). Builds initial book.
2. **All-cash already, state CASH:** `W={}`, no held names ⇒ no orders. Equity flat.
3. **Zero qualifiers in NEUTRAL/FULL:** core empty ⇒ sells of any held leaders, no buys ⇒ drifts toward
   cash. Not an error.
4. **Equity ≤ 0:** return `[]` (degenerate; Phase 3).
5. **`taper_mult` at 0.25 (deep drawdown):** book shrinks to 25% of state budget but is **never** fully
   cash via the taper (recovery preserved). Market CASH can still force full cash separately.
6. **Stop-out then immediate re-qualification:** blocked by `g_stop_block` for `STOP_COOLDOWN_DAYS` —
   the name cannot be re-bought even if its momentum still ranks. Prevents stop/re-buy whipsaw.
7. **Drift to ≥ 0.28 between rebalances:** Phase 7 forces a rebalance; Phase 8/11 trim to ≤ `NAME_CAP`.
   Concentration never reaches 0.30 for even one day, let alone 5.
8. **Sleeve present but FULL gross would exceed 1.45× beta:** Phase 8.4 clamp scales all weights down.
9. **Many positions + a crash forcing CASH:** all sells in one cycle; if > 45, sells take priority and
   are emitted first (a full liquidation of ≤ ~7 names is far under 45 anyway).
10. **Re-call same day after a rebalance:** same-day guard prevents double-trading; only trailing stops
    may fire.

---

## 17. Complexity analysis

Let `P` = `|LEADER_POOL|` (~36) + `|INDEX_REF|` + `|SLEEVE|` ≈ 40; `L` = bars per ticker (~220);
`H` = held names (≤ ~7).

- Feature computation: `O(P · L)` ≈ 40·220 ≈ 8.8k float ops per cycle (each SMA/ret/vol is `O(L)` or
  `O(n)`; computed a constant number of times per ticker).
- Ranking sort: `O(P log P)` ≈ trivial.
- Order generation: `O(P + H)`.
- **Total per cycle:** `O(P · L)` — a few tens of thousands of float operations. **Well under the 5 s
  limit** (sub-millisecond in practice).
- **Memory:** `O(P · L)` for the input bars (provided by the harness) + `O(P + H)` strategy state.
- **No network, no I/O, no third-party imports.** Pure stdlib arithmetic.

---

## 18. Developer verification checklist

A second developer should be able to tick **every** box and, doing so, match the reference behavior.

**Parameters & universe**
- [ ] All §1 constants present with the exact values; none computed at runtime.
- [ ] `LEADER_POOL`, `SLEEVE`, `INDEX_REF`, `BETA` exactly as §2.
- [ ] The agent never buys any 3× ETF and never buys QLD/SSO outside FULL.

**Features**
- [ ] `vol` uses **population** stdev × √252; returns `UNDEF` if `< n+1` bars.
- [ ] `ret(C,k)` uses `C_N / C_{N−k} − 1` and needs `≥ k+1` bars.
- [ ] `momentum_score` = `0.5·ret42 + 0.3·ret21 + 0.2·trend_gap`; `UNDEF` if any part is `UNDEF`.
- [ ] `date_of` truncates `ts` to 10 chars; all date math is string/lexicographic.

**State machine**
- [ ] Priority order is exactly CASH-trigger → cooldown → FULL-conditions → NEUTRAL.
- [ ] `brake_fired` = `ret(QQQ,3) < −0.05` OR `vol(QQQ,10) > 0.50`.
- [ ] FULL requires **all six** sub-conditions (fast SMA, slow SMA+band ×2, breadth, vol).
- [ ] Entering CASH sets `g_cooldown = 3`; FULL is impossible while `g_cooldown > 0`.

**Sizing & exposure**
- [ ] Core weights are rank-linear (`N−rank+1`), normalized to `core_budget`, capped at 0.26, **no
      redistribution**.
- [ ] `CORE_FULL = 0.67` + sleeve `0.33`; `CORE_NEUTRAL = 0.85`, no sleeve.
- [ ] All target weights multiplied by `taper_mult ∈ {1.0,0.5,0.25}` (pure function of `dd`).
- [ ] Beta-gross clamp at 1.45 applied after sizing.
- [ ] Verify: no target weight > 0.26; dollar sum ≤ 1.0; beta-gross ≤ 1.45.

**Drawdown taper**
- [ ] `taper_mult` recomputed each cycle from current `dd` — **no latch**.
- [ ] Taper floor is **0.25, never 0** (recovery preserved); only state=CASH reaches full cash.
- [ ] `g_peak_equity` updated to include current equity **before** computing `dd`.

**Trailing stops & cooldowns**
- [ ] `g_pos_high` per held name = running max close; stop when close `< high·0.92`.
- [ ] Stop-out forces a full sell **immediately** (bypasses rebalance gate) and sets `g_stop_block=3`.
- [ ] `g_cooldown` and `g_stop_block` decrement **only on a new trading day**.
- [ ] Stop-blocked names are excluded from candidate selection.

**Cadence & ordering**
- [ ] `do_rebalance` fires on: first call, cadence ≥ 3 days, de-risk transition, `taper<1.0`, or drift.
- [ ] **Same-day guard:** no second full rebalance on one `current_date`.
- [ ] Sell-before-buy: spendable = `cash + 0.98·proceeds`; buys never exceed spendable.
- [ ] Buys iterate descending target weight; quantities are integer floors; dead-band `0.03·equity`.
- [ ] `g_last_rebalance_date` set only when a full rebalance emits ≥1 order.
- [ ] Orders capped at 45, sells prioritized.

**Hygiene & safety**
- [ ] Missing `SPY`/`QQQ` ⇒ liquidate + return (Phase 0).
- [ ] Duplicate positions aggregated; `last_prices` fallbacks applied in order.
- [ ] Whole function cannot raise (outer guard returns `[]`); no divide-by-zero.
- [ ] `g_last_seen_date` / `g_prev_state` updated on every non-Phase-0 return path.

**Behavioral smoke tests (expected outcomes)**
- [ ] Calm uptrend, low vol ⇒ state FULL, holds ~5 leaders + QLD/SSO, beta-gross ≈ 1.33×, no breach.
- [ ] SPY/QQQ break 50-SMA ⇒ state CASH next cycle, full liquidation, cooldown set.
- [ ] QQQ −6% in 3 days ⇒ `brake_fired` ⇒ CASH immediately.
- [ ] A single held name −9% from its high ⇒ stopped out that cycle even if no rebalance is due.
- [ ] Early +X% then −7% from peak ⇒ `taper_mult=0.5`, book halved, **not** locked (re-expands on recovery).
- [ ] Admission (no QLD/SSO present) ⇒ core-only, beta-gross ≤ 1.0, concentration ≤ 0.26 ⇒ passes.

---

*Blueprint complete and frozen. Two implementations that satisfy §18 will be functionally identical.
**Implementation (writing `decide()`) is the next, separate step** — to be validated with
`python preview.py`, `python selfcheck.py`, and a ±20% parameter sweep.*
