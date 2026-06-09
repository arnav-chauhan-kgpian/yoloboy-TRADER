# Strategy Landscape — Builderr Trading v0

> Quant-research memo. **No code, no implementation, no parameters-as-recipe.** This is the
> strategic map: where the edge is, where the field is weak, what wins the *specific* game we're
> playing, and which single strategy family maximizes P(finish #1).
>
> Companion to [`ENVIRONMENT_ANALYSIS.md`](ENVIRONMENT_ANALYSIS.md). Read that first for the mechanics.

---

## 0. The one thing that decides this competition

We are not scored on return. We are scored on **Calmar = annualized return ÷ max drawdown**, over a
**30-day** window, and then **re-run on hidden out-of-sample windows that include stress**.

Two structural facts follow, and they dominate every design choice:

**Fact 1 — Calmar is convex in drawdown. The denominator is the whole game.**
Cutting max drawdown from 4% → 2% *doubles* your Calmar. Adding 2% of return when you're already
up 3% only lifts it ~60%. Drawdown reduction has a **convex** payoff; return has a **linear** one.
Over a short 30-day window where one bad three-day stretch sets the entire denominator, **the
agent with the smoothest equity curve and a small positive return beats the agent with the
biggest return and a normal-sized drawdown.** Everyone trading the same 20 liquid names will earn
similar gross returns; the winner is decided in the denominator.

**Fact 2 — The hidden rerun is an anti-luck filter that specifically kills aggressive bots.**
To be #1 you must (a) place top-3 in Round 1 *and* (b) survive a re-run on fresh calm+stress
windows you never saw. A bot that wins Round 1 by being lucky-long in a tech bull will meet a
stress window in the rerun and post a drawdown that destroys its Calmar. **The rerun rewards the
agent whose drawdown is low in *every* regime, not the one whose return was high in *one*.**

> **Corollary (the thesis of this memo):** Maximizing P(#1) ≠ maximizing expected return. It means
> minimizing max drawdown *conditional on staying net-positive*, **consistently across all
> regimes**. The objective is **capital preservation with a small, reliable positive drift** — not
> alpha generation. Win the denominator, harvest a little numerator, and never blow up on the rerun.

One guard rail to respect: the engine sets `Calmar = 0` when drawdown ≈ 0 (a pure-cash bot scores
**zero, not infinity**). So the target is not "no risk" — it's **minimal, controlled risk that
still captures a small positive return.** The sweet spot is a max drawdown in the low single digits
with a positive return: that produces an enormous Calmar precisely because the denominator is tiny.

---

## 1. Weaknesses of the existing agents and house bots

Every bot in this repo is readable competitor/reference source. Their flaws define the gaps to exploit.

| Bot | Design | Core weakness for *this* game |
|---|---|---|
| `baseline.py` | Buy-and-hold 4 ETFs, no risk-off | No drawdown control at all → eats the full market drop. Preview already shows ~30% MDD in the COVID window. Dead on the denominator. |
| `momentum_v1.py` | Static AI basket + 10% TQQQ, buy-once | Fair-weather, ~1.4× gross, zero de-risking. A vol spike = a blow-up. Worst-case rerun candidate. |
| `ai_momentum.py` (house "aggressive") | AI basket, gated TQQQ, XLP/XLU ballast | Still ~1.25× and tech-concentrated. The ballast is too small to move the denominator; TQQQ adds drawdown faster than return over 30 days. |
| `example_sector_rotation.py` | Faber top-4 sectors + SMA risk-off | **Concentration:** only 4 sectors at ~24% each → idiosyncratic sector drawdown drives MDD. Slow 50-day SMA lags every fast drop. Rotates into reversals. |
| `example_vol_target.py` | Inverse-vol on 5 names + SMA risk-off | Better, but the 5-name core is tech-heavy and correlated; "diversification" is illusory when SPY/QQQ/SMH/XLK all fall together. Risk-off floor still 30% exposed. |
| `seed_dual_momentum.py` | Antonacci dual momentum | Monthly-ish cadence + **tick-count rebalancing bug** (see below) → barely trades at daily cadence. Binary on/off gate whipsaws on regime edges. |
| `drawdown_momentum.py` (house "bar to beat") | 3 regimes + crash brake + vol-target to 14% | **The strongest house bot — and the real benchmark.** Weaknesses: (a) 14% vol target is *too high* for a Calmar denominator game; (b) trend + crash-brake still let a fast gap-down through before the brake fires; (c) hysteresis/cooldown logic is complex → more rerun failure surface. Beatable by running a *lower* risk budget with the same architecture. |
| `agent.py` (current "Calmar Rotation Hybrid") | SMA+vol regime, defensive book, QLD/SSO overlay | The leveraged overlay (QLD/SSO) *adds* drawdown in exactly the choppy windows the rerun will throw. The risk-on book is 5 correlated mega-cap/tech names → correlated drawdown. Good skeleton, wrong risk dial. |

**Structural weaknesses shared across the field:**

1. **Tick-count rebalancing bug (latent in most reference bots).** `REBALANCE_EVERY_TICKS = 130`
   commented "~weekly at 30-min ticks" means *130 days* at daily cadence → the bot effectively
   never rebalances in a 30-day round. Any bot timing off a `decide()` counter instead of bar
   dates is silently broken. (Edge: be cadence-robust off timestamps.)
2. **Correlated "diversification."** Holding SPY+QQQ+SMH+XLK+NVDA+MSFT feels diversified but is one
   factor (US large-cap tech beta). In a drawdown they move as one → the denominator blows out.
3. **Slow risk-off (50/100-day SMA).** By the time a long SMA flips, half the drawdown has
   happened. The denominator is already set.
4. **Leverage where it hurts.** Half the entrants (`zaid` 1.45×, `mohit`, `momentum_v1`) lean on
   leveraged ETFs, which *amplify the denominator* and decay in chop — the opposite of what Calmar
   wants.
5. **Too much risk budget.** Even the good bots target ~14% vol / ~1.0× gross. In a denominator
   game that's leaving free Calmar on the table.

---

## 2. The average competitor — likely strategies

The field is visibly clustered. From the entrant source and the ANATOMY.md/START_HERE.md funnel:

- **The dominant archetype (~half the field): "AI-basket momentum + a moving-average switch."**
  Hold NVDA/AMD/MU/MRVL/AVGO/SMH (or QQQ-tech), rank by 3-month return, go to cash when QQQ/SPY
  breaks its 100-day average. This is the literal copy-paste from `ANATOMY.md` (`sumegh` is exactly
  this). High beta, tech-concentrated, slow switch.
- **The "lever it up" archetype (~quarter): leveraged-ETF tactical.** TQQQ/SOXL/UPRO gated by trend
  and vol, often VIX-based (`zaid`, `mohit`). Chasing the numerator.
- **The "I read Faber/Antonacci" archetype: sector rotation / dual momentum.** Diversified-ish,
  slower, lower beta (`sankeerth`, the references).
- **A few "clever" archetypes: pairs/stat-arb, ML, regime models** (`shyam` pairs). Low headcount,
  high overfit risk.

**What this means for us:** the crowd is fighting on the *return* axis (who picks the hottest AI
names) in a contest scored on the *risk-adjusted* axis, and most carry tech-beta concentration or
leverage that the hidden stress rerun is purpose-built to punish. **The field is long the wrong
factor for the objective.** Our edge is not a better signal — it's being the *only* smooth,
genuinely-diversified, low-drawdown bot in a field of fair-weather beta.

---

## 3. Which styles fail in which regime

The rerun samples calm **and** stress. A winning style cannot have a fatal regime. The matrix:

| Style | Bull (sustained up) | Bear (sustained down) | Sideways / chop | Vol spike / crash |
|---|---|---|---|---|
| **Pure momentum / trend** | ✅ strong return | ⚠️ ok if risk-off fires; ❌ if SMA lags | ❌ **whipsaw death** — buys highs, sells lows, slippage bleed | ❌ **lags the gap** — drawdown set before the brake fires |
| **Mean reversion / pairs** | ⚠️ fades the winners → underperforms | ❌ **catches falling knives** → unbounded loss | ✅ **best here** — harvests oscillation | ❌❌ **catastrophic** — averages down into a crash → max MDD |
| **Sector rotation** | ✅ ok | ⚠️ concentrated sector risk | ❌ rotates into reversals; turnover bleed | ❌ slow; 4-name concentration gaps |
| **Leveraged ETF tactical** | ✅✅ best return | ❌ amplified loss, possible DQ | ❌ **decay grinds you down** | ❌❌ **blow-up / DQ risk** |
| **Buy & hold beta** | ✅ return | ❌ full drawdown | ⚠️ flat | ❌ full drawdown |
| **Vol-targeted defensive trend** | ⚠️ caps upside (that's fine) | ✅ de-risks | ✅ small book rides it out | ✅ brake + low budget = small MDD |
| **Minimum-variance / low-vol tilt** | ⚠️ lags the bull (fine for Calmar) | ✅ defensive names hold up | ✅ low turnover, low bleed | ✅ lowest-beta sleeve = smallest gap |
| **Risk-parity / diversified multi-asset** | ⚠️ modest | ✅ uncorrelated sleeves cushion | ✅ balanced | ✅ diversification caps the gap |
| **ML / regime model** | ✅ if regime ∈ training | ❌ if regime ∉ training | ❌ overfit noise | ❌❌ **out-of-sample = the rerun's kill shot** |

**Reading the matrix:** every *return-seeking* style has at least one ❌❌ (catastrophic) cell —
and the rerun guarantees that cell gets sampled. The only styles with **no catastrophic regime**
are the three at the bottom: **vol-targeted defensive trend, minimum-variance, and risk-parity.**
Those are the survivors. The winner lives in their intersection.

---

## 4. Ten candidate strategy families

Distinct families, from most aggressive to most defensive:

1. **Leveraged-ETF tactical (numerator chasing).** TQQQ/SOXL/UPRO gated by trend + vol. Max return, max denominator, DQ-prone.
2. **Concentrated AI/large-cap momentum.** The ANATOMY recipe. Rank a tech basket, hold the top few, SMA switch to cash. High beta, single-factor.
3. **Cross-sectional sector rotation (Faber).** Top-N of 11 sectors by momentum + market filter.
4. **Dual momentum (absolute + relative, Antonacci).** Gate on market trend, rotate within winners, defensive sleeve when off.
5. **Volatility-targeted broad-beta trend.** SPY/QQQ scaled to a vol target with an SMA risk-off — the `vol_target`/`drawdown_momentum` family.
6. **Mean-reversion / pairs / short-horizon stat-arb.** Fade short-term dislocations between correlated names; harvest chop.
7. **Minimum-variance / low-volatility factor tilt.** Hold the lowest-volatility, lowest-beta names (staples, utilities, healthcare, low-vol large caps), inverse-vol or min-variance weighted. Optimize for the smoothest curve.
8. **Risk-parity / diversified multi-sleeve.** Allocate equal *risk* (not dollars) across uncorrelated sleeves — equities, defensives, and (if reliably present) bonds/gold — rescaled to a low portfolio-vol target.
9. **Defensive adaptive-exposure trend with a fast crash brake, run at a LOW risk budget.** The `drawdown_momentum` architecture (3 regimes + fast multi-day/vol brake + inverse-vol sizing) but deliberately dialed to a *low* vol target and *small* gross, with broader diversification to kill idiosyncratic drawdown. **Capital-preservation-first.**
10. **Cash-plus minimal-risk harvester.** Mostly cash + a tiny, highly-diversified risk sleeve sized so max drawdown stays in the low single digits while a small positive carry accrues. The "pure Calmar denominator" play (must avoid the `MDD≈0 → Calmar=0` guard by keeping the sleeve non-trivial).

---

## 5. Ranking the families

Scored 1–5 (5 = best) on the four axes, plus a **composite weighted for P(#1)**. Because the
objective is winning a Calmar contest *with a stress rerun*, the composite weights **Hidden-rerun
survivability and Expected Calmar highest, then Robustness, then Simplicity** (simplicity matters
as a *proxy* for not overfitting, not as an end in itself).

| # | Family | Exp. Calmar | Robustness | Simplicity | Rerun survival | **Composite (P#1)** |
|---|---|:--:|:--:|:--:|:--:|:--:|
| 9 | **Defensive adaptive-exposure trend, low budget + diversified** | **5** | **5** | 3 | **5** | **🥇 4.8** |
| 7 | Minimum-variance / low-vol tilt | 4 | 5 | 4 | 5 | 🥈 4.5 |
| 8 | Risk-parity / diversified multi-sleeve | 4 | 5 | 3 | 5 | 🥉 4.4 |
| 10 | Cash-plus minimal-risk harvester | 4 | 4 | 5 | 4 | 4.1 |
| 5 | Vol-targeted broad-beta trend | 3 | 4 | 4 | 4 | 3.6 |
| 4 | Dual momentum (Antonacci) | 3 | 3 | 4 | 3 | 3.1 |
| 3 | Sector rotation (Faber) | 3 | 2 | 4 | 2 | 2.5 |
| 2 | Concentrated AI/large-cap momentum | 3 | 2 | 4 | 2 | 2.4 |
| 6 | Mean reversion / pairs | 2 | 1 | 2 | 1 | 1.5 |
| 1 | Leveraged-ETF tactical | 4* | 1 | 3 | 1 | 1.6 |

\* Leveraged tactical has *high variance* of Calmar — it can post the single best number in a lucky
calm-bull (which is why naive entrants chase it) but its expectation across the regime distribution
is poor and its rerun survival is near-zero. High ceiling, catastrophic floor. In a single-draw
contest that's a gamble; across Round-1 + rerun it's a losing bet.

**Why the top cluster (9, 7, 8) separates from the pack:** all three have **no catastrophic
regime** (§3) and all three attack the **denominator** rather than the numerator. They differ in
*how* they keep the curve smooth — adaptively cutting exposure (9), statically holding low-beta
names (7), or balancing risk across uncorrelated sleeves (8). #9 wins because it combines the
denominator discipline of low-vol with an **active brake**, so it preserves the small positive
drift in calm windows (avoiding the Calmar=0 trap) *and* collapses risk fastest in stress.

---

## 6. Final recommendation

> **Strategy family #9 — Defensive, vol-targeted, broadly-diversified adaptive-exposure trend,
> run at a deliberately LOW risk budget, with a fast multi-signal crash brake.**
>
> In one sentence: *take the house "bar to beat" architecture, but win the denominator — diversify
> beyond tech beta, target a low portfolio volatility, brake on fast price/vol signals rather than
> slow moving averages, never use leverage, and size so the worst 30-day drawdown stays in the low
> single digits while a small positive drift accrues.*

**The design intent (conceptual, not implementation):**

- **Objective = minimize max drawdown subject to staying net-positive**, not maximize return.
- **Exposure is a dial, not a switch.** Three states — risk-on (modest gross), soft-risk-off
  (small defensive sleeve), hard-stress (near-cash) — sized continuously by realized volatility so
  a vol spike auto-cuts the book before any brake even fires.
- **Diversify the risk-on book across genuinely different return drivers**, not six flavors of
  large-cap tech. The aim is to lower *portfolio* variance via low cross-correlation, because
  correlated holdings are what blow out the denominator in a drawdown.
- **Brake on fast signals** (multi-day price drops, short-window vol explosions) that lead the slow
  SMAs — the denominator is set in the first three days of a crash, so the brake must be faster
  than the crowd's 50/100-day filters.
- **No leveraged ETFs, ever.** Over a 30-day Calmar window they contribute more to the denominator
  (and decay) than to the numerator.
- **Low turnover** (date-based cadence + dead-bands) to avoid slippage bleed and whipsaw, and to
  stay cadence-robust against the unknown tick frequency.
- **Keep the moving parts few.** Every extra knob is a way to overfit Round 1 and die on the rerun;
  the simplicity here is *robustness insurance*, deliberately chosen.

**Why this maximizes P(#1) specifically (not just expected return):**
The field is crowded with high-beta, fair-weather, sometimes-levered bots fighting on the return
axis. Their Calmar is high-variance and their rerun survival is low. By being the **smoothest,
most diversified, lowest-drawdown bot that still earns a small positive drift**, we (a) win the
convex denominator game in Round 1, and (b) are one of the few bots that *survives* the stress
rerun with its Calmar intact — and survival is the precondition for the #1 slot. We're not trying
to have the best month; we're trying to have the **least-bad worst-case across every regime they
can throw**, which is exactly what the two-stage Calmar-plus-rerun scoring rewards.

---

## 7. Why it beats each named alternative

**vs. Pure momentum** — Momentum earns more numerator in a trend but pays it back in the
denominator: it whipsaws in sideways markets (buy high / sell low + slippage) and lags vol spikes
(drawdown is set before the signal flips). Our family *uses* trend for direction but **subordinates
it to a volatility/drawdown budget and a fast brake**, so we keep momentum's upside capture while
removing its two failure regimes. In a Calmar contest, controlled exposure > raw signal strength.

**vs. Sector rotation** — Rotation concentrates into ~4 sectors (idiosyncratic drawdown), reacts
slowly, and rotates *into* reversals at turning points. It's a less-diversified, slower cousin of
momentum with the same denominator problem and worse rerun survivability. Our broader, vol-weighted
diversification produces a structurally smoother curve for the same expected return.

**vs. Mean reversion** — Mean reversion has the **single worst tail for this objective**: in a
crash it averages *down into* the move, manufacturing exactly the deep drawdown Calmar punishes
most, and it loses unboundedly in any sustained trend. It posts a pretty Sharpe in calm chop and
then gets annihilated the moment the rerun samples a stress window. It optimizes the wrong moment
of the distribution.

**vs. Leveraged-ETF approaches** — Leverage *multiplies the denominator*, decays in chop, and
risks the >50% DQ / auto-flatten. Its Calmar is high-variance with a catastrophic floor; it can
flukishly top a single calm-bull Round 1 but is the *first* thing the stress rerun eliminates.
P(#1) across two stages with a leveraged bot is near-zero. We reach for return through
*diversification and exposure timing*, which lifts Calmar without inflating the denominator.

**vs. Machine-learning approaches** — The rerun is *designed* to kill ML: it's an explicit
out-of-sample test on regimes that, by construction, aren't in any training set. With only ~220
daily bars × ~21 delivered tickers, there is nowhere near the data to fit a generalizing model;
any ML bot will overfit Round 1's window and revert to noise (or worse) on the rerun — that's the
exact Phase A↔B inconsistency the organizers flag for review. It's also impractical under the
stdlib-only constraint. A transparent, few-parameter rules engine generalizes *better* and is
*auditable* (relevant given the top-10 human code read). Simplicity here is not a limitation — it's
the robustness edge.

---

## 8. Caveats and open questions that could move the call

The recommendation is robust to most of the §C unknowns in `ENVIRONMENT_ANALYSIS.md`, but three
could sharpen it:

1. **If the Round-1 window turns out to be a strong, low-vol bull** (and the rerun weighting is
   light), a higher risk budget would have scored better. Mitigant: the rerun explicitly re-tests
   on stress, so over-defensiveness is the *safe* error and over-aggression is the *fatal* one —
   asymmetric, and we take the safe side. We can tune the risk budget *up to the point where
   worst-regime drawdown still stays small*, capturing more bull upside without surrendering the
   denominator.
2. **If defensive assets (TLT/GLD) are not reliably in `market_state`** (admission ships only ~21
   tickers, defensives = XLP/XLU/XLV/XLE), the "diversify across asset classes" lever is weaker and
   we lean more on the *exposure dial* (cash as the diversifier) and low-beta equity sectors. The
   recommendation survives; the multi-asset variant (#8) is the one that would degrade.
3. **If the Calmar `MDD≈0 → 0` guard bites** (we end *too* defensive and barely trade), we'd score
   zero. The design must keep the risk sleeve non-trivial enough to register a real (small)
   drawdown and a real positive return — the target is *low* drawdown, not *no* drawdown.

**Verdict:** Across the plausible range of these unknowns, **family #9 maximizes the probability of
finishing #1.** It is the only family that is simultaneously a strong expected-Calmar bot, has no
catastrophic regime, and is built to survive the exact thing that decides the title — the
out-of-sample stress rerun.

*Next step (separate task): translate this into a concrete `decide()` design — signal set, exposure
states, sizing rule, brake triggers, and cadence — then validate on `preview.py` and stress-stub
windows. No code until that design is locked.*
