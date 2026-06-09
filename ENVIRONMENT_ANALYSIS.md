# Builderr Trading Template — Environment Analysis

> Technical reconnaissance of the repository. **No strategy is proposed here.** This documents
> the contract, the simulator, the scoring, and every hidden assumption that could change how a
> winning agent must be built. Read this before writing a single line of `decide()`.

Date of analysis: 2026-06-08. Round 1 is **live** (June 2 – July 2, 2026); leaderboard `as_of` 2026-06-05.

---

## 1. Repository structure

37 tracked files. Grouped by role:

### Documentation (read-only, human-facing)
| File | Role |
|---|---|
| `START_HERE.md` | 5-minute plain-English onboarding; the "go to cash" pitch. |
| `AGENT_BRIEF.md` | The canonical spec to paste into an AI — contract, rules, scoring, traps. |
| `ANATOMY.md` | "Anatomy of a strong bot" — the 4-move recipe (momentum + risk-off + vol sizing + caps). |
| `README.md` | Authoritative rules, constraints table, universe, scoring stages, submission paths. |
| `LICENSE` | MIT. |

### The thing you submit
| File | Role |
|---|---|
| `agent.py` | **Your submission.** Currently contains a complete reference strategy, "Calmar Rotation Hybrid". |

### Local test harness (no engine, no network — what you actually run)
| File | Role |
|---|---|
| `preview.py` | **Primary local check.** Replays your bot over 3 public sample windows; prints metrics + PASS/FAIL on the admission safety bar. Pure stdlib. |
| `selfcheck.py` | Data-free smoke test on synthetic bars + a committed-secret scanner. |
| `strategy_selftest.py` | Strategy-specific unit tests for the *current* `agent.py` (caps/regime/contract). Imports `agent`. |
| `sample_regimes.json.gz` | The 3 public windows `preview.py` uses (calm 2021, selloff 2021, COVID 2020). |

### Private-engine harness (cannot run from this repo — needs the `builderr` package + Polygon key)
| File | Role |
|---|---|
| `local_test.py` | 1-week SVB-2023 mini Phase A at 30-min ticks. Reference only. |
| `full_test.py` | All 3 hidden Phase A regimes at 30-min ticks. Reference only. |
| `fairness_tests.py` | Published source of the engine's determinism/fairness tests (audit transparency). Reference only. |

### Live leaderboard infra (runs in CI, not part of the builder workflow)
| File | Role |
|---|---|
| `live_runner.py` | Runs the whole field on live yfinance daily bars from `ROUND_START`; writes `leaderboard.json`. |
| `.github/workflows/leaderboard.yml` | GitHub Action: runs `live_runner.py` hourly while US market open, commits the board. |
| `leaderboard.json` | Current standings (generated artifact). |
| `build_universe.py` | Builds the frozen `universe.json` (top ~1000 by dollar-volume). Run once at round open. |
| `universe.json` | The frozen tradeable universe (1000 tickers). |

### Reference / house bots (read, run, beat)
| File | Family |
|---|---|
| `baseline.py` | Equal-weight buy-and-hold of 4 ETFs (simplest admitted bot). |
| `drawdown_momentum.py` | House "bar to beat" — drawdown-first vol-managed momentum, 3 regimes + crash brake. |
| `seed_dual_momentum.py` | House all-weather — Antonacci dual-momentum sector rotation. |
| `ai_momentum.py` | House aggressive — AI basket + gated TQQQ overlay. |
| `momentum_v1.py` | Earlier dogfood — static AI basket buy-and-hold w/ TQQQ. |
| `example_sector_rotation.py` | Reference — Faber sector momentum + SMA risk-off. |
| `example_vol_target.py` | Reference — inverse-vol sizing + SMA risk-off. |

### Round-1 entrant bots (competitors, scored on the live board)
`opu_agent.py`, `robert_agent.py`, `mohit_agent.py`, `zaid_agent.py`, `sumegh_agent.py`,
`shyam_agent.py`, `harsimran_agent.py`, `sankeerth_agent.py`, `siddu_agent.py`, `rohit_agent.py`.
These are real submissions registered in `live_runner.py`'s `FIELD`. They are readable competitor source.

### Misc
`.gitignore`, `requirements.txt` (empty — "no third-party packages required by agent.py").

---

## 2. Purpose of every file

Covered in §1. Key takeaways:

- **Only `agent.py` matters for submission.** Everything else is scaffolding, examples, or infra.
- **`preview.py` is the only test you can actually run** with no setup (stdlib only). `local_test.py`/`full_test.py`/`fairness_tests.py` require the private `builderr` engine you don't have.
- The **competitor source is fully visible** — you can read every other entrant's exact logic in this repo.

---

## 3. Exact `decide()` interface

```python
def decide(market_state, portfolio_state, cash) -> list[dict]:
    # Return orders, e.g. [{"ticker": "SPY", "side": "buy", "quantity": 10}]
    # Return [] to do nothing.
    return []
```

- Called **once per decision tick**. In `preview.py` and `live_runner.py` that is **once per trading day**. In the private engine (`full_test`/`local_test`) it is called every `tick_interval_minutes` (30 in those scripts; the docs say admission may run at **1-minute ticks**, Phase B "finer"). **The cadence is not fixed and not knowable from inside `decide()`.** (See §12, assumption #1 — this is the single most dangerous hidden detail.)
- **Return value:** a `list` of order dicts. Each order: `{"ticker": str, "side": "buy"|"sell", "quantity": number}`. Empty list = no action.
- Order validation (from `preview.py` / `live_runner.py`): an order is accepted only if `side in ("buy","sell")`, `float(quantity) > 0`, and `ticker` is present in the provided `market_state`/universe. Malformed orders are silently dropped (and count as an error in `preview.py`'s `errors`, which would FAIL the "runs clean" gate).
- **Quantity is in shares** (not dollars, not weights). `float` is allowed (fractional shares fill in preview/live).
- Orders are capped: `preview`/engine enforce **≤ 50 trades/day**; `agent.py` itself slices `orders[:45]` defensively. The current `agent.py`'s `orders_to_rebalance` returns `orders[:45]`.
- **Module-level globals persist across calls within one regime/process**, and are **reset between regimes** (`preview.py` re-imports the module fresh per regime; the engine runs each regime in its own process). Do not assume globals survive across regimes.

---

## 4. Structure of `market_state`

```python
market_state = {
    "SPY": [bar, bar, ..., bar],   # oldest first
    "QQQ": [ ... ],
    ...
}
```

Each `bar` is a dict:
```python
{"ts": "2021-06-01", "open": 419.9, "high": 421.2, "low": 418.0, "close": 420.4, "volume": 48357400}
```

- Bars are **daily**, **oldest-first**, prices appear **split/dividend-adjusted** (`live_runner` uses `auto_adjust=True`; sample data is adjusted).
- `ts` is an ISO date string `"YYYY-MM-DD"` in the public data. (The engine `Tick` uses `datetime` timestamps / pandas DataFrames internally; the daily-bar dict shape is what `decide()` receives in the documented contract.)
- **History length ≈ 220–340 daily bars.** Contract says "≈220 trading days (~10 months) including a pre-regime warmup so even 200-day signals work from tick one." Sample data actually ships **~340 total bars per ticker** with **~64 eval days + ~276 warmup days** (e.g. calm_uptrend: SPY history 2020-04-27 → 2021-08-30, eval window 2021-06-01 → 2021-08-31).
- **Critical alignment difference between the two runners:**
  - `preview.py`: `market_state` includes every bar **up to AND INCLUDING today** (`b["ts"] <= date`). You see today's close, then your order fills at **next day's open**.
  - `live_runner.py`: `market_state` includes bars **strictly before today** (`b["ts"] < date`); `last_prices` = prior close. Your order fills at **today's open**.
  - Both avoid lookahead, but the data offset and fill timing differ. **A bot tuned on `preview.py`'s "include today" view will see a one-bar-shifted world on the live board.**
- **Not every universe ticker is present every tick.** Sample regimes contain only **21 tickers** (`AAPL, JPM, KRE, META, MSFT, NVDA, QQQ, SMH, SOXL, SPY, TQQQ, XLC, XLE, XLF, XLI, XLK, XLP, XLRE, XLU, XLV, XLY`). The full universe is 1000, but `market_state` only carries the tickers the engine fed for that regime. **Always `market_state.get(t)` defensively and skip missing names.** Tickers absent from `market_state` cannot be traded.

---

## 5. Structure of `portfolio_state`

```python
portfolio_state = {
    "cash": 100000.0,
    "positions": [
        {"ticker": "SPY", "quantity": 12.0, "avg_cost": 420.4},
        ...
    ],
    "last_prices": {"SPY": 421.0, "QQQ": 350.2, ...},
}
```

- `cash` — spendable cash. **Equals the `cash` third argument** (convenience copy).
- `positions` — list of held lots; **only `quantity > 0` positions are included** (zero/closed positions are dropped). Each has `ticker`, `quantity`, `avg_cost`.
- `last_prices` — mark price per ticker. In `preview.py` this is **today's close**; in `live_runner.py` it is the **prior close**. Use it to compute equity / drift; don't assume it's the same reference price your order will fill at.
- **Equity is not given directly — you compute it:** `cash + Σ quantity × last_price`. Every reference bot does this manually.
- There is **no margin/buying-power field, no fees field, no realized-P&L field.** What you see is what there is.

---

## 6. Available assets / tickers

- **Full universe:** `universe.json` — exactly **1000 tickers**, top US names by trailing dollar-volume, **frozen at round open** (same for everyone, stable all round). Top names: NVDA, TSLA, MU, SNDK, MSFT, AAPL, AMZN, AMD, GOOGL, META, AVGO, INTC, PLTR, ... plus ETFs.
- **Anything off the list is silently ignored.**
- **Leveraged ETFs present in the universe:** `TQQQ, SOXL, UPRO, SPXL, QLD, SSO` (and `SOXX` which is **not** leveraged). The full beta tables:
  - **3×:** TQQQ, SOXL, UPRO, SPXL, TNA, FAS, TECL, LABU, CURE, DRN, UDOW, NAIL
  - **2×:** QLD, SSO, DDM, ROM, UWM, AGQ
  - **1×:** everything else.
  - ⚠️ Note: TNA, FAS, TECL, LABU, CURE, DRN, UDOW, NAIL, DDM, ROM, UWM, AGQ appear in the beta table but **are not all in `universe.json`** — they only matter if they're actually tradeable. Verify membership before relying on one.
- **ETFs always force-included** in `build_universe.py`: SPY QQQ DIA IWM VTI VOO, XLK XLF XLE XLV XLI XLY XLP XLU XLRE XLC XLB, SMH SOXX IGV ARKK ARKQ XBI IBB KRE ITB XHB, GDX GLD SLV TLT HYG USO VNQ EEM EFA, + leveraged sleeve.
- **But in admission, only ~21 tickers per regime are actually delivered** (see §4). Do not assume GLD/TLT/VTI are present in `market_state` during admission — the sample regimes don't include them. The defensive sleeve that's *reliably* present in sample data is XLP/XLU/XLV/XLE.

---

## 7. Available historical lookback

- **~220 daily bars guaranteed by contract; ~276–340 in the shipped sample data.** Enough for a 200-day SMA "from tick one" because of the warm-up prefix.
- Lookback is **daily resolution** in the data dict regardless of tick cadence.
- The eval/scoring window is **separate** from the warm-up: the warm-up bars exist so signals are valid on day 1 of the eval window, but P&L is only counted across the eval window (~30 days in admission regimes; 30 days live; ~64 in the longer sample windows).
- You **cannot** see beyond the latest provided bar (no lookahead). Querying external data for the regime period = disqualification (§11).

---

## 8. Existing example agents and their logic

### House / reference bots
| Bot | Logic | Caps |
|---|---|---|
| `baseline.py` | Buy-and-hold 25% each of SPY/QQQ/XLK/XLV on first tick, then hold forever. No risk-off. | 25%/name, ~1.0× gross. |
| `drawdown_momentum.py` ("bar to beat") | 3 regimes (on/soft/hard) via SPY+QQQ 100-day trend + index 6-mo momentum + fast crash brake (QQQ −6%/3d, −8%/5d, or 10-day vol > 70%). Risk-on: cross-sectional momentum top-6, inverse-vol sized, vol-targeted to 14% annual, gross ≤ 1.0×. Hard stress → ~10% gross in XLP/XLU. Hysteresis band ±1%, 5-tick rebalance, 3% dead-band, cooldown after stress. **No leveraged ETFs.** | 18%/name, gross ≤ 1.0×. |
| `seed_dual_momentum.py` | Antonacci dual momentum: absolute gate (SPY > 50-day SMA) then relative (rank 9 sectors by 60-day return, top 5 equal-weight). Gate off → XLP/XLU/XLV/XLE/XLI. Weekly-ish rebalance via tick counter, 27% drift force. | ~20%/name, ~1.0×. |
| `ai_momentum.py` | Fixed AI basket (QQQ/SMH/NVDA/MSFT/AAPL/META + 10% TQQQ + XLP/XLU ballast). TQQQ gated by QQQ 20>50 SMA trend and halved if QQQ vol > 30%; freed weight → ballast. | ~1.25× calm / ~0.95× stressed. |
| `momentum_v1.py` | Static AI basket incl. 10% TQQQ, buy once and hold. ~1.40× gross. No risk management. | 25% max, 1.40×. |
| `example_sector_rotation.py` | Faber: SPY < 50-day SMA → defensive (XLP/XLU/XLV @25% + cash); else top-4 of 11 sectors by 60-day return @ ~24%. | 24%/name, ~1.0×. |
| `example_vol_target.py` | Inverse-20-day-vol weights across SPY/QQQ/SMH/XLK/XLV, capped 28%; total exposure 0.95× (risk-on) or 0.30× (SPY < 50-day SMA). | 28%/name, ≤0.95×. |

### Current `agent.py` — "Calmar Rotation Hybrid"
- **Risk-on gate:** SPY & QQQ both above 50-day SMA **and** QQQ 20-day annualized vol < 35%.
- **Risk-off book:** XLP 24% / XLU 24% / XLV 20% / XLE 12%, rest cash.
- **Risk-on book:** score `RISK_CANDIDATES` (broad + sector + 7 mega-caps) by `0.55·mom60 + 0.25·mom20 + 0.20·trend_gap − 0.15·vol20`; take top 5 with positive score.
- **Tactical overlay:** small QLD 11% / SSO 7% only when QQQ 20>50 SMA, QQQ 20-day momentum > 0, QQQ vol < 28%, and both ETFs present. Never TQQQ/SOXL.
- **Caps:** per-name ≤ 24% (`MAX_WEIGHT`), drift force-rebalance > 27%, beta-adjusted gross scaled to ≤ 1.35×, 5-**day** rebalance cadence (date-based, not tick-count — robust to cadence), min trade 1.5% of equity.

### Round-1 entrants (competitor source, fully readable)
- `opu` — 6-month momentum (skip 1mo), top-4 equal-weight @22%, cash when < 4 trending; ~0.88× gross, no leverage.
- `robert` — drawdown-first trend + cross-sectional momentum + vol scaling + equity-curve guard; ≤1.35× gross.
- `mohit` — aggressive AI/Nasdaq + capped TQQQ; crash brake de-levers; ≤1.4× gross, 24% cap.
- `zaid` — 3× momentum (TQQQ/SOXL/UPRO) + TLT risk-off on SPY<20d or VIX≥25; VIX-scaled; ≤1.45× gross, 25% cap. **Uses VIX (external?) and leverage heavily.**
- `sumegh` — straight ANATOMY.md recipe: NVDA/AMD/MU/MRVL/AVGO/SMH, top-4 above 50-MA, QQQ 100-MA safety switch.
- `shyam` — relative-momentum pairs (buy outperformer of correlated pairs) + SPY 50-day risk-off.
- `harsimran` — adaptive exposure, signal-quality/vol/correlation sizing, often < 1.0× gross.
- `sankeerth` — defensive trend, inverse-vol 13% vol target, 3 brakes (hard/panic/trend), GLD/TLT defensive.
- `siddu` — large-cap momentum w/ 0.5% hysteresis band, blended 75/15-day momentum, top-6 @28%, ≤1.30×.
- `rohit` — clone of house drawdown-momentum regime stack + AI tilt, no leverage, 18% cap.

**Observation:** the field clusters hard around "momentum + SMA risk-off + caps." Differentiation is in regime detection, sizing, and crash-brake speed, not in novel signals. The live board (as of 2026-06-05, ~3 days in) shows **almost everyone slightly negative** (−2% to −8.6%); only `sankeerth` is marginally positive (+0.02%). Early-window noise dominates.

---

## 9. Simulator constraints and limits

From `README.md`, `AGENT_BRIEF.md`, and the actual code in `preview.py` / `live_runner.py`:

| Rule | Limit | Enforced how |
|---|---|---|
| Side | **Long-only** | Sell clamped to held qty; no shorting. |
| Gross beta-adjusted exposure | **≤ 1.5× equity** | `Σ |position$| × beta / equity`. Sustained breach > 60s → auto-flatten + DQ. Preview FAILs if peak > 1.5×. |
| Position concentration | **< 30% per ticker** | Breach only if held **≥ 30% for MORE than 5 consecutive days** (`preview`: `worst_streak <= 5` passes). A brief excursion is fine. |
| Trades / day | **≤ 50** | Excess rejected. |
| Minimum hold | **≥ 60 seconds** | Excess rejected (irrelevant at daily cadence). |
| `decide()` runtime | **≤ 5 s / call** | Tick errors out; you keep going (engine), or counts as error (preview). |
| Catastrophe | **> 50% drawdown = blow-up** | Fails admission. |
| LLM | optional, **bring your own key** | Not needed; keeps it about ideas. |

### Fill mechanics (from `preview.py` / `live_runner.py`)
- **Fills at the OPEN** (next day in preview; same day in live), **not** the close you decided on.
- **Slippage:** 5 bps for 1× equities, **10 bps for leveraged ETFs**. Buys fill at `open × (1+slip)`, sells at `open × (1−slip)`.
- **No commissions / no spread beyond slippage.**
- **No margin / no borrowing cash:** a buy is **clamped to available cash** (`if cost > cash: qty = cash/fill`). ⚠️ **This means total dollar exposure can never exceed 1.0× via plain stock.** The *only* way to reach the 1.5× beta-adjusted gross cap is leveraged ETFs (which carry 2×/3× beta but cost real cash). Gross-in-dollars ≤ 1.0× always; gross-in-beta can go to 1.5× only through TQQQ/SOXL/QLD/SSO/etc.
- **Orders execute sequentially**; earlier buys consume cash before later ones. Order matters when cash-constrained. `agent.py` deliberately sells before buying and only counts 98% of sell proceeds as spendable to avoid over-ordering.
- Fractional share quantities are allowed and fill.
- **Determinism guaranteed** (`fairness_tests.py`): same code + same data → identical fills; same order → identical fill regardless of which agent sent it; slippage is a function of the instrument only.

---

## 10. How `preview.py` works

1. Loads `agent.py` (or a filename arg) via `importlib`, requires a `decide` attribute.
2. Loads `sample_regimes.json.gz` → 3 regimes (`calm_uptrend`, `moderate_selloff`, `vol_spike_snapback`). Each has `eval_start`, `eval_end`, and compact bars `[ts,o,h,l,c,v]` expanded to bar dicts.
3. **Per regime, fresh module reload** (resets globals). Iterates eval dates:
   - Fill yesterday's pending orders at **today's open** (± slippage, cash-clamped).
   - Mark-to-market at **today's close** → equity curve.
   - Compute risk telemetry: peak gross (beta-adjusted), per-ticker concentration streak.
   - Build `market_state` = all bars `ts <= today`, `portfolio_state`, call `decide()`. New orders become **pending for next day's open**.
4. Computes `ret`, max drawdown, Sharpe (×√252), Calmar (annualized ret / MDD).
5. **Safety bar** (the only admission gate): runs clean (no errors), peak gross ≤ 1.5×, no concentration streak > 5 days, worst drawdown < 50%. ALL four must pass.
6. Prints verdict. Exit code 0 if admitted, 1 otherwise.

**What preview does NOT do:** it is not your official score. Admission runs centrally on **hidden** regimes (the public windows are illustrative). The 30-day live forward test is what ranks you.

⚠️ **Windows encoding bug:** `preview.py`'s final verdict line prints a `✓` (U+2713). On a Windows `cp1252` console this raises `UnicodeEncodeError` **after** the safety-bar PASS/FAIL block prints — so the checks display fine but the script crashes on the last line. Run with `PYTHONUTF8=1` (or `set PYTHONIOENCODING=utf-8`) to see the clean verdict. The bug is cosmetic; the gating logic already ran.

---

## 11. How scoring is calculated

Three stages (`README.md` §Scoring):

1. **Admission (instant, on submit).** Runs your bot across **3 hidden 30-day historical regimes** (shapes: fast sector-contagion crash; slow rate-hike downtrend; vol spike + leveraged-unwind snapback — real dates 2022–2024, hidden). **Pass = (no leverage/concentration breach) AND (no >50% drawdown) AND (runs without fatal error).** It is a **safety screen, NOT a skill gate.** You also get a free "robustness profile" (Sharpe/DD/return per regime).

2. **Round 1 — live forward test (June 2 – July 2, 2026, 30 days).** Admitted agents trade the shared paper sandbox, same fills for everyone. **Ranked by Calmar = annualized return ÷ max drawdown.** "+10% with a −2% dip beats +30% with a −25% dip." This is the competition.

3. **Held-out rerun (anti-luck).** Top finishers re-run on **fresh windows they never saw** (calm + stress). Confirms skill over luck. Lookahead cheaters get caught here via Phase A↔B correlation checks.

**Calmar exact form** (from `preview.py`, the same metric code the engine reuses):
- `annualized_return = (1 + total_return)^(252/days) − 1`
- `max_drawdown = max peak-to-trough fractional drop on the equity curve`
- `Calmar = annualized_return / max_drawdown` (0 if MDD ≈ 0).

⚠️ **The live `leaderboard.json` is sorted by raw `ret`, not Calmar.** `live_runner.py` does `rows.sort(key=lambda r: r["ret"], reverse=True)` and only reports equity/P&L/return/trades — **no drawdown or Calmar column at all.** The displayed board ordering is **not** the official ranking metric; the note says "the final winner is risk-adjusted." Don't optimize for the visible board.

**Prizes:** Top 3 by Phase B Calmar (surviving rerun) split $2,000 ($1200/$500/$300). Top 5 LinkedIn spotlight. #1 runs a real $100k Nasdaq book.

### Lookahead / anti-cheat (hard rule)
- **No lookahead bias = the one absolute rule.** Network access is allowed (news, alt-data, your own LLM/server), but querying data that reveals the regime period's future at submission time = DQ.
- Caught via: (1) top-10 human code reads (`requests.get("yahoo/SPY/2023-*")` inside the backtest = DQ + public postmortem), (2) Phase A↔Phase B Sharpe correlation check, (3) surprise fresh-regime reruns.

---

## 12. Hidden assumptions & implementation details that affect strategy design

These are the non-obvious things that will silently break a naive bot. Ranked by impact.

1. **⚠️ Tick cadence is unknown and inconsistent across harnesses — tick-count rebalancing is a trap.**
   `preview.py`/`live_runner.py` call `decide()` **once per day**. `full_test.py`/`local_test.py` call it every **30 minutes** (~13×/day). Docs say admission may be **1-minute** ticks (~390×/day) and Phase B "finer." Several reference bots use `REBALANCE_EVERY_TICKS = 130` *commented "~weekly at 30-min ticks."* **At daily cadence, 130 ticks = 130 days = the bot never rebalances during a 30-day round.** `ai_momentum.py`, `seed_dual_momentum.py`, `example_*` all carry this latent mismatch. The current `agent.py` avoids it by rebalancing on **bar dates** (`REBALANCE_EVERY_DAYS = 5`, derived from `ts`), which is cadence-robust. **Any winning bot must derive timing from bar timestamps, never from a `decide()` call counter.**

2. **⚠️ `preview` vs `live` data offset & fill timing differ.** Preview: `market_state` includes today, fills next open. Live: `market_state` excludes today, fills today's open. A signal computed on "the latest bar" refers to different days in each. Validate that your logic is correct under *both* — and that you never implicitly depend on seeing the bar you fill on.

3. **⚠️ Dollar gross is capped at 1.0× by the no-borrow rule.** You cannot reach 1.5× beta-gross with plain equities — only via leveraged ETFs, which cost full cash and carry 2×/3× beta + 10 bps slippage + volatility decay. The 1.5× "leverage" headroom is effectively a *leveraged-ETF* allowance, not a margin allowance. Buy-and-hold of leveraged ETFs is explicitly discouraged (decay, drawdown denominator).

4. **Concentration cap is a 5-consecutive-day tolerance, not a hard line.** You may sit ≥30% in one name for **up to 5 trading days**; only day 6+ breaches. Brief overweight from drift is safe. But in a 30-day round, 5 days is ~1/6 of the window — don't lean on it.

5. **Admission `market_state` is sparse (~21 tickers), not 1000.** Strategies that assume GLD/TLT/VTI/IWM/specific names are present will get empty lists for them during admission. The reliably-present defensive names in sample data are **XLP/XLU/XLV/XLE**. Always `.get()` and degrade gracefully; never index `market_state[t]` blindly.

6. **Calmar's denominator is max drawdown → protecting drawdown beats chasing return.** Over a short 30-day window, one −8% day can dominate Calmar. The house "bar to beat" (`drawdown_momentum.py`) is built entirely around this: be fully invested only in calm uptrends, de-risk hard and fast on stress. A fair-weather high-return bot can still be *admitted* but loses the ranking.

7. **Short window (30 days) → high variance, signal lag matters.** A 50/100/200-day SMA reacts slowly; by the time it flips, the move is half over. Crash brakes in the field act on 3–5 day returns and 10-day vol. The held-out rerun penalizes overfit fast-reacting parameters — there's tension between responsiveness and robustness.

8. **Globals reset between regimes/processes, persist within.** Don't carry state you expect across regimes. Don't rely on "first tick" detection that assumes a single continuous run.

9. **Slippage asymmetry rewards low turnover.** Every round-trip costs ≥10 bps (1×) / 20 bps (lev). High-frequency rebalancing bleeds Calmar. Dead-bands (2–3% of equity) and min-trade thresholds are standard in the field for this reason.

10. **`avg_cost` and `last_prices` references differ from fill prices.** Equity/drift computed from `last_prices` (close/prior-close) will not match what your order actually fills at (next/this open ± slip). Size with a safety margin; `agent.py` discounts sell proceeds to 98% spendable.

11. **The leaderboard is generated content, not the eval.** `live_runner.py` + the GitHub Action produce `leaderboard.json` on yfinance data. It is honest but (a) sorted by raw return, (b) uses the live same-day-open fill model, (c) refreshes intraday. It is **not** how you're officially scored. Treat it as a sanity signal only.

12. **No transaction costs beyond slippage; no overnight/borrow/financing costs** — even for leveraged ETFs (their decay only shows up via the underlying price path in the bars, not as an explicit charge).

13. **Determinism is guaranteed** — there's no randomness in fills, no queue priority, no partial-fill uncertainty. Same code → same score. This means there is **no luck in execution**; all variance is in the market path and your logic.

14. **`requirements.txt` is empty by design** — `agent.py` must run on **stdlib only** for the no-dep workflow (the engine won't pip-install your deps for the simple path). If you need packages, that's the endpoint-mode / private-engine path, not the default.

15. **Universe contains look-alikes:** `SOXX` (1× semis ETF) is in the universe but is **not** the 3× `SOXL`; `SSO`/`QLD` are 2× while `SPY`/`QQQ` are 1×. Mis-tagging beta = accidental leverage breach.

---

## Questions that must be answered before building a winning agent

These are genuinely unresolved from the repo alone and materially change design. Grouped by how blocking they are.

### A. Blocking — wrong answer here breaks the bot or DQs it
1. **What is the actual `decide()` call cadence in admission and in Phase B live?** Daily? 1-minute? 30-minute? The docs conflict (contract says "daily-resolution in admission; finer in Phase B"; README says 1-min; test scripts use 30-min). This determines whether *any* tick-count logic is valid and how rebalance/brake timing must be expressed.
2. **Regardless of tick cadence, are the bars in `market_state` always daily, or do they become intraday at finer cadences?** If Phase B feeds 1-minute bars, every SMA/momentum/vol calc silently changes meaning. (The agent docstrings claim "daily bars" but that's an assumption, not confirmed for live.)
3. **Which fill model does the official engine use — next-tick open (preview) or same-tick open (live_runner) or something else (VWAP/mid)?** And is the decision made on the prior close or the current bar? This changes whether your latest-bar signal is actionable.
4. **Is the 1.5× cap measured on beta-adjusted gross only, or is dollar gross independently capped?** Confirm that leveraged ETFs are the *only* path to >1.0× and that the no-borrow cash clamp applies identically in the real engine.
5. **Exactly when does a concentration breach trigger — strictly >5 consecutive days at ≥30%, or any touch of 30%?** preview says >5 consecutive; README prose says "for any 5 trading days." Confirm the real engine's rule and whether it's measured at close, intraday, or per tick.

### B. High-impact — changes strategy selection
6. **What is the full set of tickers actually delivered in `market_state` during admission and live?** The sample ships only 21. Is the live `market_state` the full 1000, a liquid subset, or only names the agent has "touched"? If defensive assets like GLD/TLT/IWM aren't reliably present, the risk-off book must be built from what is.
7. **Is the Round-1 ranking pure Calmar, or Calmar with tie-breaks / minimum-trade / minimum-activity filters?** (e.g. does a flat all-cash bot with 0 drawdown get an undefined/huge Calmar, or is it floored/excluded?) The `Calmar = 0 if MDD≈0` rule in preview would *penalize* a perfectly flat bot — is that the official behavior?
8. **How is annualization handled for a 30-day window, and is return measured close-to-close over exactly the eval window?** `(1+ret)^(252/days)` is extremely sensitive to a 30-day sample — confirm `days` = trading days in window.
9. **What counts as acceptable external data vs lookahead?** Is using a *live* VIX feed (as `zaid_agent` does) allowed in Phase B (where "now" is genuinely now), but a DQ in admission (where "now" is a 2022–2024 backtest)? Where exactly is the line, and is VIX even available in `market_state` or must it be fetched?
10. **Do the admission regimes and Phase B use split/dividend-adjusted prices (auto_adjust)?** Confirmed for `live_runner`; assumed for sample. If the real engine uses raw prices, momentum/SMA signals shift around ex-div dates.

### C. Operational / verification
11. **Can the official admission/Phase-B engine be run locally at all** (is the `builderr` package obtainable, or is `preview.py` truly the only feedback loop pre-submission)? If preview is the only loop, how do we validate behavior at the real cadence?
12. **Is `agent.py` the required entry filename, and must `decide` be module-level?** (preview/live import `decide` from the file; confirm no class/instance wrapper is needed.)
13. **How many revisions are allowed and does each reset admission?** Docs say "revise up to 4 times" (START_HERE) vs "resubmit anytime before your cohort locks" (README) — which governs, and does a resubmit re-run admission on the same hidden regimes?
14. **What is the exact starting cash and is it always $100,000?** (All code uses 100k; confirm Phase B isn't scaled.)
15. **Are fractional shares actually honored by the official engine,** or only integer quantities? (preview/live allow floats; the reference bots all emit integers via `//`.) This affects small-account sizing precision.
16. **Does the 60-second min-hold or 50-trades/day cap interact with a finer live cadence** in a way that throttles intraday rebalancing? At 1-min ticks, 50 trades/day is ~1 trade per 8 minutes — does that constrain a brake that wants to flatten everything at once (could be >50 orders if many positions)?

---

*End of analysis. No strategy proposed — that's the next step, once the questions above are resolved.*
