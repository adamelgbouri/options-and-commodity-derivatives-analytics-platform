# OCDAP — Options & Commodity Derivatives Analytics Platform

> A professional-grade commodity derivatives pricer built in Python.  
> Covers 65+ commodities across 8 asset classes with live futures data, six pricing engines, and an interactive Streamlit dashboard.

![Python](https://img.shields.io/badge/Python-3.10%2B-blue?style=flat-square)
![Streamlit](https://img.shields.io/badge/Streamlit-1.32%2B-red?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)

---

## Live Demo

Run locally:
```bash
streamlit run ocdap.py
```

---

## Features

- **65+ commodities** — Energy, Metals, Agriculture, Base Metals, Freight, Carbon & Environmental
- **Dual data source routing** — Yahoo Finance (grouped download) or TradingView (tvdatafeed)
- **Black-76 vanilla options** — European call/put with full Greeks, price/delta surface, put-call parity verification
- **Asian options (Monte Carlo)** — arithmetic and geometric average, Kemna-Vorst analytical benchmark, convergence analysis
- **Crack spread options (Kirk 1995)** — 3-2-1, simple crack, jet crack, spark spread
- **Calendar spread options** — Kirk on term structure, all M_i−M_j pairs across the curve
- **Commodity swaps** — fixed-for-floating NPV, break-even rate, DV01, cashflow schedule, sensitivity
- **Barrier options** — knock-in / knock-out down/up, daily Monte Carlo (252 steps), price vs barrier level
- **Volatility surface** — parametric smile/skew model, 3D interactive surface, term structure
- **Expiry-aware tickers** — automatically skips expired contracts, always downloads the true front month

---

## Products Covered

| Tab | Product | Model | Primary user |
|---|---|---|---|
| 📉 Vanilla Options | European call/put on futures | Black-76 | Options Desk |
| 🎲 Asian Options | Arithmetic / geometric average-price | Monte Carlo + Kemna-Vorst | Physical Hedger |
| 🏭 Crack Spread | Option on refining margin $F_2 - F_1$ | Kirk (1995) | Margin Hedger |
| 📅 Calendar Spread | Option on curve slope $M_\text{near} - M_\text{far}$ | Kirk on term structure | Spread Trader |
| 🔄 Commodity Swaps | Fixed-for-floating swap | Discounted cashflows | Physical Hedger |
| 🚧 Barrier Options | Knock-in / knock-out | Daily Monte Carlo | Options Desk |
| 📊 Vol Surface | Parametric implied vol surface | Black-76 + skew | Options Desk |
| 📈 Forward Curve | Live futures curve | Cost-of-carry extension | All users |

---

## Data Sources

| Source | Commodities | Notes |
|---|---|---|
| Yahoo Finance | WTI, Brent, NG, Gold, Silver, Copper, Grains, Softs... | Free, no credentials |
| TradingView | Jet CIF NWE, LME metals, TTF, Coal, Carbon, Freight... | Optional credentials |

---

## Commodities Covered

| Family | Commodities | Source |
|---|---|---|
| Energy | WTI, Brent, Natural Gas, RBOB, Heating Oil, Gasoil | Yahoo Finance |
| Energy+ | Jet CIF NWE, TTF, NBP, Coal API2/API4, Uranium, Carbon EUA | TradingView |
| Metals | Gold, Silver, Copper, Platinum, Palladium | Yahoo Finance |
| Base Metals | LME Copper, Aluminum, Zinc, Nickel, Lead, Tin, Cobalt | TradingView |
| Agriculture | Corn, Wheat, Soybeans, Sugar, Coffee, Cocoa, Cotton | Yahoo Finance |
| Agriculture+ | Soybean Oil/Meal, Oats, OJ, Cattle, Hogs, Lumber | Yahoo Finance |
| Freight | Capesize, Panamax, Supramax, VLCC | TradingView |
| Carbon | EU EUA, UK UKA, California CCA, RGGI | TradingView |

---

## Dependencies

| Library | Purpose |
|---|---|
| `numpy` | Monte Carlo simulation, array operations |
| `pandas` | Forward curve DataFrames, cashflow schedules |
| `scipy` | `norm.cdf` for Black-76, `brentq` for implied vol inversion |
| `yfinance` | Live futures prices (NYMEX, COMEX, CBOT, ICE) |
| `streamlit` | Interactive browser dashboard |
| `plotly` | Zoomable charts, 3D volatility surface |

```bash
pip install numpy pandas scipy yfinance streamlit plotly
pip install git+https://github.com/StreamAlpha/tvdatafeed.git  # optional, for TradingView
```

---

## Theory

**Black-76** is the industry standard for European options on commodity futures:

$$C = e^{-rT}\left[F \cdot N(d_1) - K \cdot N(d_2)\right], \quad d_1 = \frac{\ln(F/K) + \tfrac{1}{2}\sigma^2 T}{\sigma\sqrt{T}}$$

Unlike Black-Scholes, it prices options on **futures prices** $F$ directly — observable, liquid, and free of cost-of-carry assumptions.

**Kirk (1995) approximation** converts a two-asset spread option into an effective single-asset Black-76 problem:

$$\sigma_{\text{Kirk}} = \sqrt{\sigma_1^2 + \left(\frac{F_2}{F_2 + Ke^{-rT}}\right)^2 \sigma_2^2 - 2\rho\,\sigma_1\sigma_2\,\frac{F_2}{F_2 + Ke^{-rT}}}$$

**Asian option** arithmetic average (Monte Carlo):

$$C_{\text{Asian}} = e^{-rT}\,\mathbb{E}\!\left[\max\!\left(\frac{1}{N}\sum_{i=1}^{N}S_{t_i} - K,\;0\right)\right]$$

The geometric average has a closed-form solution (Kemna-Vorst 1990) used as analytical benchmark:

$$\sigma_g = \sigma\sqrt{\frac{2N+1}{6(N+1)}}, \qquad F_g = F \cdot e^{b_g T}$$

**Commodity swap** NPV and break-even fixed rate:

$$\text{NPV} = \sum_{i=1}^{N} e^{-rT_i}(F_i - K)\cdot\text{notional}, \qquad K^* = \frac{\sum_i e^{-rT_i} F_i}{\sum_i e^{-rT_i}}$$

**Barrier option** (Down-and-Out call, daily Monte Carlo):

$$C_{\text{KO}} = e^{-rT}\,\mathbb{E}\!\left[\max(S_T - K,\,0)\,\mathbf{1}_{\left\{\min_{0 \le t \le T} S_t > B\right\}}\right]$$

See [`docs/OCDAP_Technical_Report.pdf`](docs/OCDAP_Technical_Report.pdf) for the full mathematical write-up.

---

## Key Design Choices

**Expiry-aware tickers** — `build_tickers()` skips any contract whose estimated expiry (20th of the prior month) has passed. On April 28 2026, this generates `CLM26` (Jun, true M1) instead of `CLJ26` (Apr, expired March 20) — ensuring Yahoo Finance always returns live prices, not stale data from expired contracts.

**Kirk vs Margrabe** — Kirk (1995) handles non-zero strikes $K \neq 0$, which is the standard case for crack spread options. Margrabe's exact formula only applies when $K = 0$.

**Monte Carlo seed** — all MC engines use `numpy.random.seed(42)` for reproducibility. Increase `n_paths` in the sidebar (up to 100,000) to reduce the confidence interval.

**Forward curve extension** — when live data is shorter than the requested curve length, OCDAP extends using the cost-of-carry model anchored at the last downloaded price, ensuring all pricers always have a full curve.

---

## Relationship with CFCAP

OCDAP and CFCAP are complementary platforms:

| | CFCAP | OCDAP |
|---|---|---|
| Purpose | Forward curve analytics | Derivatives pricing |
| Models | PCA, Schwartz-Smith, convenience yield | Black-76, Kirk, Asian MC, Barrier MC |
| Output | 51 trading signals, curve analysis | Option prices, Greeks, swap NPV |
| Typical user | Curve trader, risk manager | Options desk, structurer, hedger |

---

## License

MIT — © 2026 Adam El Gbouri

---

*Built with Python · Yahoo Finance · TradingView*
