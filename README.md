# builderr Trading Agent — submission

Submission for the **builderr Trading Agent Leaderboard** (Round 1: June 2 – July 2, 2026).

The agent is implemented in **`agent.py`** as a single function, `decide()`. It is self-contained:
no network, no LLM, no API keys, Python standard library only.

## Validate locally

No engine, no install, no keys:

```bash
python preview.py     # runs the agent across 3 real sample windows; prints the admission safety bar (PASS/FAIL)
python selfcheck.py   # quick smoke test (well-formed orders) + committed-secret scan
```

`preview.py` reproduces the *shape* of the admission report (clean run, leverage cap, concentration
cap, no blow-up). It is not the official score — admission runs centrally on hidden regimes — but a
clean PASS is a strong predictor of admission.

## The contract

```python
def decide(market_state, portfolio_state, cash) -> list[dict]:
    return [{"ticker": "SPY", "side": "buy", "quantity": 10}]
```

| Argument | Shape |
|---|---|
| `market_state` | `{ticker: [bar, ...]}` — recent **daily** bars per ticker, oldest first (≈220 trading days, including a pre-regime warmup). Each bar: `{ts, open, high, low, close, volume}`. |
| `portfolio_state` | `{cash, positions: [{ticker, quantity, avg_cost}], last_prices: {ticker: price}}` |
| `cash` | Convenience copy of `portfolio_state["cash"]`. |
| **return** | List of orders. Each: `{ticker, side: "buy"\|"sell", quantity: float}`. Empty list = no action. |

`decide()` is called once per decision interval (daily-resolution in admission; finer in Phase B live).

## Constraints (auto-enforced)

| Rule | Limit | Breach action |
|---|---|---|
| Side | Long-only | Order rejected |
| Gross beta-adjusted exposure | ≤ 1.5x equity | Sustained breach > 60s → auto-flatten + DQ |
| Position concentration | < 30% per ticker for any 5 trading days | Sustained breach → auto-flatten + DQ |
| Trade rate | ≤ 50 trades/day | Excess rejected |
| Min hold | ≥ 60s | Excess rejected |
| `decide()` runtime | ≤ 5s per call | Tick errors out |

**Beta multiples** (for the leverage cap): 3x — TQQQ, SOXL, UPRO, SPXL, TNA, FAS, TECL, LABU, CURE,
DRN, UDOW, NAIL · 2x — QLD, SSO, DDM, ROM, UWM, AGQ · 1x — everything else.

**No lookahead bias** is the one absolute rule. This agent is signal-driven from the provided daily
bars only — no external data, no future information.

## Universe

The tradeable set is the **top ~1000 US names by liquidity**, frozen at round open. The exact frozen
list is in [`universe.json`](universe.json); anything outside it is silently ignored.
