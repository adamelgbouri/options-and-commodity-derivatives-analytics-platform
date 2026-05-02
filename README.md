# OCDAP — Options & Commodity Derivatives Analytics Platform

> Six industrial pricing engines. 65+ commodities. Live futures data.  
> From vanilla Black-76 to barrier Monte Carlo — all in a single Python file.

![Python](https://img.shields.io/badge/Python-3.10%2B-blue?style=flat-square)
![Streamlit](https://img.shields.io/badge/Streamlit-1.32%2B-red?style=flat-square)
![Numpy](https://img.shields.io/badge/NumPy-Monte%20Carlo-orange?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)

---

## What is OCDAP?

OCDAP is a Python application that prices commodity derivatives on a **live forward curve** downloaded from Yahoo Finance or TradingView. It implements six pricing engines used daily by energy trading desks, covering vanilla options, Asian average-price options, crack spread options, calendar spread options, commodity swaps, and barrier options.

Every pricer is connected to the same forward curve, ensuring consistency across all products and maturities.

---

## Live Demo

```bash
git clone https://github.com/your-repo/ocdap
pip install -r requirements.txt
streamlit run ocdap.py
```

---

## Pricing Engines

| # | Product | Model | Typical desk user |
|---|---|---|---|
| 1 | **Vanilla Options** | Black-76, full Greeks, implied vol | Options desk |
| 2 | **Asian Options** | Arithmetic MC + Kemna-Vorst geometric benchmark | Physical hedger |
| 3 | **Crack Spread Options** | Kirk (1995) approximation | Margin hedger / refiner |
| 4 | **Calendar Spread Options** | Kirk on term structure, all $M_i - M_j$ pairs | Spread trader |
| 5 | **Commodity Swaps** | NPV, break-even rate $K^*$, DV01, cashflow schedule | Physical hedger |
| 6 | **Barrier Options** | KI/KO Down/Up, daily Monte Carlo (252 steps) | Options desk |

Plus a **Vol Surface** tab (parametric smile + skew model, 3D interactive) and a **Forward Curve** tab with live data for all 65+ commodities.

---

## Features

- **Expiry-aware tickers** — `build_tickers()` automatically skips expired contracts and always downloads the true front month. On April 28 2026 this generates `CLM26` (Jun delivery, exp. May 20) instead of `CLJ26` (Apr delivery, expired March 20) — ensuring Yahoo Finance returns live prices, not stale data.
- **Put-call parity verification** — Black-76 call and put are verified against $C - P = e^{-rT}(F-K)$ on every computation. Displayed as `✓ verified` in the dashboard.
- **MC convergence chart** — Asian option tab shows the MC price vs number of paths on a log scale, with the Kemna-Vorst geometric price as a lower bound.
- **Kirk vs Margrabe** — Kirk (1995) handles $K \neq 0$ strikes, which is the standard case for crack spread options. The effective Kirk vol is recomputed for every strike and maturity pair.
- **Forward curve extension** — when live data is shorter than the requested curve length, OCDAP extends using cost-of-carry anchored at the last downloaded price.
- **Synthetic warning banner** — a clear `⚠ SYNTHETIC PRICES` banner appears before Run Analysis, so pre-download prices are never confused with live market data.
- **Dual data source** — Yahoo Finance (free, no credentials) for exchange-listed futures; TradingView (optional) for LME metals, TTF gas, carbon, freight.

---

## Commodities Covered

| Family | Count | Commodities | Source |
|---|---|---|---|
| Energy | 7 | WTI, Brent, Natural Gas, RBOB, Heating Oil, Gasoil, Jet CIF NWE | Yahoo / TradingView |
| Metals | 5 | Gold, Silver, Copper, Platinum, Palladium | Yahoo Finance |
| Base Metals | 9 | LME Copper, Aluminum, Zinc, Nickel, Lead, Tin, Cobalt, Steel HRC, Iron Ore | TradingView |
| Agriculture | 8 | Corn, Wheat CBOT/KC HRW, Soybeans, Sugar, Coffee, Cocoa, Cotton, OJ | Yahoo Finance |
| Agriculture+ | 7 | Soybean Oil/Meal, Oats, Live/Feeder Cattle, Lean Hogs, Lumber, Milk | Yahoo Finance |
| Energy+ | 8 | TTF, NBP, Coal API2/API4, Uranium, Propane, Ethanol, Carbon EUA | Yahoo / TradingView |
| Freight | 4 | Capesize BCI, Panamax BPI, Supramax BSI, VLCC TD3C | TradingView |
| Carbon | 4 | EU EUA, UK UKA, California CCA, RGGI | TradingView |
| **Total** | **65+** | | |

---

## Theory

**Black-76** — the industry standard for European options on commodity futures. Unlike Black-Scholes, it prices options directly on the observable futures price $F$, with no cost-of-carry adjustment needed:

$$C = e^{-rT}\bigl[F\cdot N(d_1) - K\cdot N(d_2)\bigr], \qquad d_1 = \frac{\ln(F/K)+\tfrac{1}{2}\sigma^2 T}{\sigma\sqrt{T}}$$

**Kirk (1995)** — converts a two-asset spread option into an effective single-asset Black-76 problem. The adjusted volatility is:

$$\sigma_\text{Kirk} = \sqrt{\sigma_1^2 + \left(\frac{F_2}{F_2+Ke^{-rT}}\right)^2\!\sigma_2^2 - 2\rho\,\sigma_1\sigma_2\,\frac{F_2}{F_2+Ke^{-rT}}}$$

Accurate for $\rho > 0.70$, which always holds for crude vs refined products (typical $\rho \approx 0.85-0.95$).

**Asian option** — arithmetic average has no closed form. OCDAP uses Monte Carlo with the Kemna-Vorst (1990) geometric price as a lower bound (AM-GM inequality guarantees arithmetic $\geq$ geometric):

$$C_\text{Asian} = e^{-rT}\cdot\frac{1}{M}\sum_{j=1}^{M}\max\!\left(\frac{1}{N}\sum_{i=1}^{N}S_{t_i}^{(j)}-K,\;0\right)$$

**Commodity swap** NPV and break-even fixed rate $K^*$:

$$\text{NPV} = \sum_{i=1}^{N}e^{-rT_i}(F_i - K)\cdot\text{notional}, \qquad K^* = \frac{\sum_i e^{-rT_i}F_i}{\sum_i e^{-rT_i}}$$

**Barrier option** (Down-and-Out call, daily Monte Carlo at 252 steps/year):

$$C_\text{KO} = e^{-rT}\,\mathbb{E}\!\left[\max(S_T-K,\,0)\cdot\mathbf{1}_{\left\{\min_{0\le t\le T}S_t > B\right\}}\right]$$

See [`docs/OCDAP_Technical_Report.pdf`](docs/OCDAP_Technical_Report.pdf) for the full mathematical write-up.

---

## Dependencies

| Library | Role |
|---|---|
| `numpy` | Monte Carlo simulation, array operations |
| `pandas` | Forward curve DataFrames, cashflow schedules |
| `scipy` | `norm.cdf` for Black-76, `brentq` for implied vol inversion |
| `yfinance` | Live futures prices — NYMEX, COMEX, CBOT, ICE |
| `tvdatafeed` | TradingView data — LME, TTF, carbon, freight (optional) |
| `streamlit` | Interactive browser dashboard |
| `plotly` | Zoomable charts, 3D volatility surface |

```bash
pip install numpy pandas scipy yfinance streamlit plotly
pip install git+https://github.com/StreamAlpha/tvdatafeed.git  # optional
```

---

## Relationship with CFCAP

OCDAP and CFCAP are complementary platforms designed to be used together:

| | [CFCAP](https://aeg-cfcap.streamlit.app/) | OCDAP |
|---|---|---|
| **Purpose** | Forward curve analytics | Derivatives pricing |
| **Models** | PCA, Schwartz-Smith 3-factor, convenience yield | Black-76, Kirk, Asian MC, Barrier MC |
| **Output** | 51 trading signals, curve analysis | Option prices, Greeks, swap NPV, DV01 |
| **Data output** | CSV snapshots, daily scheduler | Live forward curve, cost-of-carry extension |
| **Typical user** | Curve trader, risk manager | Options desk, structurer, margin hedger |

CFCAP analyses the shape, structure and dynamics of the forward curve.  
OCDAP takes that curve and prices the derivatives written on it.

---

## License

MIT — © 2026 Adam El Gbouri

---

*Built with Python · Yahoo Finance · TradingView*
