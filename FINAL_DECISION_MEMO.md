# FINAL DECISION MEMO — Three Candidates, One Pick

> Three complete, internally-coherent strategy candidates for Builderr v0, spanning the
> variance/survival frontier, followed by probability estimates and **one** decisive
> recommendation. No code. No hedging.
>
> Builds on [`FINAL_STRATEGY_SPEC.md`](FINAL_STRATEGY_SPEC.md) (robust) and
> [`CRITIQUE_AND_REDESIGN.md`](CRITIQUE_AND_REDESIGN.md) (convex). This memo resolves the tension
> between them.

---

## The frame that decides everything: this is a TWO-gate tournament

You are not scored once. You are scored **twice in series**: Round-1 live Calmar (rank among ~1000),
**then** a held-out rerun on unseen calm+stress windows that the organizers explicitly built to
*"confirm skill, not luck."* Final standing = (Round-1 rank) **filtered by** (rerun replication).

That product structure is the whole game:

> **P(final #1) = P(post a top Round-1 Calmar) × P(that result replicates on a fresh stress window).**

A maximally convex bet maximizes the **first** factor and minimizes the **second**. A robust bet does
the reverse. The winner of a two-gate tournament is whoever maximizes the **product** — which lives in
the **middle** of the frontier, not at either end. Hold that thought; it is why I pick B.

Also note the real payoff: **top-3 split $2,000 ($1200/$500/$300), top-5 get a spotlight, #1 trades a
real $100k book.** The money and the prestige are a *top-3* phenomenon with a #1 jackpot. The rational
objective is "maximize P(top-3), with P(#1) as the prize inside it" — not "P(#1) or bust."

---

## CANDIDATE A — "FORTRESS" (maximum robustness / rerun survival)

**1. Core philosophy.** Never blow up; minimize drawdown in *every* regime; accept a low ceiling. Win
by attrition — be the bot still standing with a clean curve when the volatile field self-destructs.
This is the [`FINAL_STRATEGY_SPEC.md`](FINAL_STRATEGY_SPEC.md) design with its one real bug fixed (the
latched equity guard → a proportional, re-expandable taper).

**2. Portfolio construction.** ETF-only, no single stocks, no leverage. 6–8 names drawn from a
diversified pool (BROAD: SPY/QQQ · SECTOR: XLK/XLF/XLE/XLI/XLY/SMH · DEFENSIVE: XLP/XLU/XLV/XLRE),
selected by slow blended momentum (63/126-day) with an own-trend filter, spread across buckets.

**3. Position sizing.** Inverse-volatility weighting (risk-parity-lite). Per-name cap **18%**, bucket
cap **40%**. Calm names get more capital; jumpy names less. Routinely under-deployed (cash residual).

**4. Exposure policy.** Volatility-targeted to a **low 9%** annualized. Three states: RISK-ON (gross
0.4–1.0×), SOFT (0.25×), HARD (cash + tiny XLP/XLU). Gross scales *down* continuously as vol rises.

**5. Risk controls.** Fast crash brake (multi-day drop / vol explosion → cash); **proportional** gross
taper on portfolio drawdown (halve at −6%, cash at −10%, *re-expandable* — no lockout); cooldown +
hysteresis to suppress whipsaw; timestamp-based weekly cadence; immediate de-risk, deliberate re-risk.

**6. Expected return profile.** +1% to +5% in a normal month; ceiling ~6%. Flat-to-slightly-negative in
chop. Structurally low numerator.

**7. Expected drawdown profile.** 2–4% typical; ~8% bad month; **never above ~12%.** The smoothest curve
in the field.

**8. Strengths.** Admission near-certain; rerun survival near-certain; never embarrassing; lowest
denominator in the field; auditable and lookahead-clean.

**9. Failure modes.** Under-participation: in any trending month the high-numerator crowd out-Calmars
it badly (annualization amplifies their return). Sideways → near-zero return → mediocre/zero Calmar.
**It almost never wins; it almost always places respectably.**

---

## CANDIDATE B — "SPEAR" (balanced / maximum expected rank) ★ RECOMMENDED

**1. Core philosophy.** Concentrated conviction trend-following with **gated convexity** and a **single
fast parachute.** Hold the *actual market leaders*, press them when the trend is confirmed, add a
*measured* leverage kicker only when everything aligns, and cut to **100% cash hard and fast** when the
trend breaks — so the right-tail months are big **and they replicate on the rerun.** This is the
deliberate middle of the frontier: enough numerator to win, enough survival to keep the win.

**2. Portfolio construction.** Hold the **top 3–5 momentum leaders**, including **single stocks**
(NVDA/AVGO/MSFT/META/AMD and whatever the live universe's strongest movers are) *and* high-momentum
ETFs (SMH/XLK/QQQ). Selection by a **fast** blended score (≈ 0.5·42-day + 0.3·21-day momentum +
0.2·trend-gap) — fast because this is a 30-day sprint, not a multi-month hold — with an own-trend
filter (price > 50-day SMA) and a positive-breadth check on SPY/QQQ.

**3. Position sizing.** **Conviction (rank) weighting**, not inverse-vol — let the strongest name carry
the most (top name ~26%, descending), per-name cap **26%** (under the 30%/5-day rule, drift-trimmed).
3–5 names → a genuinely concentrated core (~0.9–1.0× dollar) that actually captures the leaders'
moves, instead of diluting them across 8 ETFs.

**4. Exposure policy.** Three asymmetric states off **fast** gates (20/50-day, not 100-day):
- **FULL** (SPY & QQQ above fast+slow trend, breadth positive, vol low): concentrated core **plus a
  gated 2× ETF sleeve (QLD/SSO)** sized to lift beta-gross to **~1.35×** (never the 1.5× edge — leave
  margin). The convex kicker that powers the numerator.
- **NEUTRAL** (mixed signals): concentrated core only, no leverage, ~0.7–1.0×.
- **CASH** (trend broken or vol spike): **100% cash**, immediately.

**5. Risk controls.** **One hard binary switch** — SPY/QQQ below 50-day SMA beyond a band, *or* fast
brake (R₃ < −5% / vol₁₀ > 50%) → flat to cash next open. **Per-name trailing stop** (exit a holding
that gives back > ~8% from its in-trade high) → caps single-name denominator damage. **Proportional,
re-expandable** portfolio taper (halve gross at −6% drawdown, cash at −10%) — *no latched lockout, fixed
for the continuous live run.* Drift-trim < 28%. Re-entry on confirmed resumption (2–3 day confirm —
faster than A, slower than chasing). Leverage **only** in FULL.

**6. Expected return profile.** −3% to +12%; modal good month **+5–8%**; ceiling ~15%. The fast
signals + leader concentration + leverage kicker put real numbers in the numerator, which the 8.4×
annualization then magnifies into a high Calmar.

**7. Expected drawdown profile.** 3–6% typical; ~12–15% worst case (a gap-down before the switch fires,
amplified by the leverage sleeve); rarely beyond ~18%; **never near the 50% blow-up line.** The
trailing stop + cash switch keep the denominator small in the *good* months — which is exactly when
Calmar is being set.

**8. Strengths.** Holds the assets that actually move (numerator); gated 2× convexity adds right tail
*without* the 3× decay/DQ risk; the hard parachute + trailing stop keep a small denominator **and**
make the right-tail outcomes *replicate* through the rerun; lockout bug fixed. **Best expected final
rank, best P(top-3), best FINAL P(#1) — see the decision section for why "best final #1" is B, not C.**

**9. Failure modes.** Whipsaw months (stop-out + re-entry drag knocks return toward zero); a single
gap-down through the switch with the leverage sleeve on (~15% DD); concentration in the *wrong* leader
during a sharp factor rotation; narrow-breadth reversal. Moderate blow-up risk — **rarely
catastrophic**, because leverage is capped at 1.35× / 2× and the switch is binary and fast.

---

## CANDIDATE C — "LANCE" (maximum P(#1) in isolation / accept failure)

**1. Core philosophy.** Pure convex lottery-with-a-parachute. Maximize the right tail; accept that most
months you fail. Designed to post a *monster* Calmar in a clean trend or nothing.

**2. Portfolio construction.** **1–2** strongest momentum names at max concentration (~28% each) **plus
a 3× leveraged ETF sleeve** (TQQQ/SOXL/UPRO) gated to a confirmed strong uptrend, pushing beta-gross
toward the **1.45×** cap. Essentially all-in on the single best trend.

**3. Position sizing.** Top-1/2 conviction; leverage sleeve sized to hit ~1.45× beta-gross in full
risk-on. No diversification, no inverse-vol.

**4. Exposure policy.** **Bimodal** — ~1.45× beta-gross when trend + momentum + low-vol all align, else
**100% cash.** No middle state.

**5. Risk controls.** A **single** tight trailing stop (give back ~5% → flat) + a same-bar vol/price
brake. That is the *entire* parachute. No diversification, no vol-target, no soft state. If a gap jumps
the stop, you eat the full leveraged loss.

**6. Expected return profile.** Bimodal: a large mass near 0 / negative (cash, whipsaw, stopped out) +
a thin tail of **+15% to +30%** months. Ceiling ~30%+.

**7. Expected drawdown profile.** When it works: tiny (trailing stop → MDD ~3–5% → Calmar 40–100). When
it fails: large (15–30%+ on a leveraged gap), occasionally flirting with admission/rerun limits.

**8. Strengths.** Highest possible Calmar ceiling; in a clean monotonic uptrend month it can be
*literally #1 in Round 1*.

**9. Failure modes.** Frequent. Whipsaw stops it to ~0 (Calmar ~0); leveraged gaps blow through the
stop (large DD); wrong-leader / narrow reversal; and — decisively — **its Round-1 wins are
luck-driven and the rerun is built to not replicate luck.** High P(rerun failure).

---

## Probability estimates (FINAL standing, after admission + Round 1 + rerun)

These are honest final-outcome probabilities for ~1000 entrants — **not** single-stage Round-1 numbers.
P(rerun failure) = P(a Round-1 top result fails to replicate and drops out of contention).

| Metric | A — Fortress | **B — Spear ★** | C — Lance |
|---|---|---|---|
| **P(Top 1 / #1)** | ~0.5% | **~1.6%** | ~1.1% |
| **P(Top 3)** | ~3% | **~7%** | ~4% |
| **P(Top 10)** | ~30% | **~36%** | ~12% |
| **P(Admission failure)** | ~1% | ~4% | ~10% |
| **P(Rerun failure** \* **)** | ~3% | ~14% | ~40% |

\* Conditional flavor: among entries that *reach* a top-Round-1 position, the share that fail to
replicate. C's headline ceiling is the highest in **Round 1 alone** (~3–4% chance of the top R1
Calmar), but ~40% rerun-failure collapses its **final** #1 probability *below* B's.

---

## DECISION: ship **Candidate B — "Spear."** No hedge.

**B maximizes the only quantity that pays: final P(top-3), with the best final P(#1) inside it.** Here
is the full defense, including why I reject A and C outright.

### 1. The two-gate theorem kills C — C's "max #1" is a single-stage illusion.
P(final #1) = P(top Round-1) × P(replicates on rerun). Plug the numbers:
- **C:** ~3.5% (top R1, fattest tail) × ~0.30 (survives a fresh stress window — leverage gaps, luck
  doesn't replicate, the rerun is *designed* to catch exactly this) ≈ **~1.1%.**
- **B:** ~2.2% (top R1, still strongly convex via leaders + gated 2×) × ~0.70 (concentrated-but-
  capped, hard cash switch, trailing stop → the good-month curve actually repeats) ≈ **~1.6%.**

C wins more Round-1 lotteries; **B keeps more of the wins.** Through two gates, the survival factor
dominates the ceiling factor, so **B has the higher *final* #1 probability** — and a vastly higher
P(top-3) (7% vs 4%), where the actual money is. The intuition from the critique ("go maximally convex
for #1") was answered for a *one-shot* metric. This is not one-shot. Correcting for the rerun, the
convex extreme is **over-levered for the structure.**

### 2. A is eliminated on ceiling. 
Fortress is the best *fund* and a poor *tournament entry*. Its ~6% return ceiling, after 8.4×
annualization, simply cannot out-Calmar the leader-holding field in any normal or trending month
(~85% of regime space). P(#1) ~0.5%, P(top-3) ~3%. It is engineered to place top-decile and concede
the podium. We are not here to place.

### 3. B captures ~80% of C's upside at ~35% of its failure risk.
B holds the real leaders (the numerator A refuses) and adds gated **2×** convexity (the right tail A
refuses) — so its good months are +5–8% with a small denominator, which is championship-grade Calmar.
But it caps leverage at 1.35×/2× (not 1.45×/3×), runs 3–5 names (not 1–2), and de-risks on a *binary*
switch — so its winning curve **replicates**, which is the entire point of the rerun. It sits exactly
where the two-gate product is maximized.

### 4. B's residual risks are bounded and acknowledged, not catastrophic.
Worst realistic outcome is a ~15% gap-day drawdown with the leverage sleeve on — unpleasant, far from
the 50% blow-up line, admission-safe. P(admission failure) ~4% is the price of carrying convexity;
it is worth paying for 2–3× the podium probability of A. The lockout bug that crippled the original
spec is explicitly fixed (proportional, re-expandable taper).

### 5. It fits the real payoff curve.
The prize is top-3 with a #1 jackpot. B is the **only** candidate that is simultaneously near the top
on P(#1) *and* clearly first on P(top-3) *and* respectable on P(top-10). On a probability-weighted
expected-prize basis, **B dominates A and C — it is not close.**

> **Ship Spear.** Concentrated leaders, fast trend gates, a gated 2× convex kicker in confirmed
> uptrends, one hard cash switch, a per-name trailing stop, and a re-expandable drawdown taper. It is
> built to **win the podium and replicate the win** — the exact shape the two-gate, top-3-paying
> structure rewards. A concedes the title; C donates it to the rerun. B takes it.

*Next step (separate task): implement Candidate B's `decide()` from this memo + the §-level detail,
then validate on `preview.py` and a ±20% structural-robustness sweep. No code in this document.*
