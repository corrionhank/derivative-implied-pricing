# Personal Market Watchlist & Analytics Dashboard — Project Brief

*A custom, always-on watchlist that aggregates the options, volatility, and market-expectation metrics I actually care about into one screen — to support my own trade decisions. Built on tastytrade's free real-time API.*

This brief condenses a multi-session scoping conversation for further planning. Decisions are settled unless listed under "Open Decisions."

---

## 1. What It Is

A personal analytics dashboard — think of it as a **custom watchlist** — that pulls the market info I care about into one place instead of scattered across a dozen tabs: options pricing, expected moves, volatility, skew, and the fear gauges, each shown in the context of its own history.

It isn't a predictor. It decodes what the options market has **already priced in**, so I can see where the market is and where it's expected to go, then make my own call. It informs decisions; it doesn't make them.

The whole thing is feasible — and free — because **tastytrade gives funded non-professional accounts free real-time data through its API**, including its signature metrics (IV rank, expected move, etc.) pre-computed. That's the enabler and the reason for the stack choice.

Analytical lens: tastytrade's — volatility/premium and probabilities, mechanics over prediction.

---

## 2. Design Principles

- **Aggregate, don't scatter** — one screen for everything that matters.
- **Contextualize everything** — percentile / z-score / regime, never raw numbers.
- **Inform, don't predict** — read what's priced; no buy/sell signals.
- **Least noise, most signal** — every panel earns its place; track the liquid complex, not single-name noise.
- **Transparency over a single score** — expose components; composites (e.g. fear/greed) appear as one labeled lens, never as a verdict.
- **Implied vs. realized is the spine** — the gap (IV−HV / variance risk premium) is the core cross-signal.
- **Seed history from day one** — store implied-vol history now; the deferred research layer needs history that can't be recreated later.

---

## 3. What It Displays

### Per-underlying volatility panel (the core watchlist rows)
For each liquid underlying:
- **IV Rank** & **IV Percentile** (show both — IVR distorts after a single spike; IVP is steadier in sustained regimes)
- **IVx per expiration** (~30–45 DTE)
- **30-day IV, 30-day HV, IV−HV** (the VRP)
- **Expected move** (1 SD) — daily / weekly / to next catalyst
- **Skew** (25-delta risk reversal / put-call IV skew)
- **Term structure** (front vs. back; contango / backwardation)
- **Liquidity** rating
- **Delta-based probabilities** (POP, probability of touch)

### Market-state strip — the four questions
Every panel answers one:
1. **Regime** — VIX/VIX3M term structure, VRP, HY credit-spread percentile, curve slope (2s10s / 3m10y)
2. **Stretch** — price vs. 50/200-DMA + distance percentile, realized-vol percentile, drawdown/rally percentile
3. **Priced** — expected-move cone, skew
4. **Why** — filtered, market-moving news feed (ideally clustered near the moves it explains)

### Backdrop
- **Cross-asset vol complex** — VIX, VIX1D, VVIX, MOVE (bonds), GVZ (gold), OVX (oil)
- **Fear & Greed** composite (drill-down to its 7 components)
- **Beta-weighted deltas to SPY** — net exposure view (once positions are tracked)

### Universe (proposed)
SPY/SPX, QQQ, IWM, /ES, /VX + a short set of liquid single names (TBD). Deliberately excludes illiquid single-name noise.

---

## 4. What It Models / Decodes

- **Expected-move cone** — the 1 SD range from the ATM straddle / IVx, projected to the next week / next catalyst. The signature visual; communicates the whole idea at a glance.
- **Implied vs. realized gap** (VRP / IV−HV) — the core cross-signal: is vol rich or cheap relative to what's actually happening?
- *(Phase 2)* Implied distribution via Breeden-Litzenberger (full priced shape incl. fat tail/skew); conditional base rates; tail modeling.

---

## 5. Data Sources & Cost

**Cost: ~$0 recurring.**

**Primary — tastytrade API** (via the tastyware `tastytrade` Python SDK):
- Option chains, Greeks (dxFeed stream), and the `/market-metrics` endpoint returning IVR, IVP, IVx, IV−HV, beta, liquidity — the signature metrics pre-computed.
- Free real-time for funded non-professional accounts; **$0 minimum to fund**, and ACH deposits/withdrawals are free, so the parked deposit is costless and reversible. (Unfunded → 14 days live, then 20-min delayed until funded.)
- SDK includes a 45-DTE helper; tastytrade conventions are baked in.
- Caveats: SDK is unofficial (well-maintained, no vendor SLA); dxFeed is slow for bulk *historical* backfill — fine for a focused hourly collector, painful for years-at-once. Compute own IVR alongside theirs for validation/resilience.

**Free supporting sources:**
- **yfinance** — OHLCV, indices, VIX complex (1h up to ~730 days; no native 4h → resample from 1h)
- **FRED** — rates, curve, HY credit spreads (OAS), breakevens (daily; fine for slow series)
- **vix_utils** — CBOE VIX-futures + cash term structure (daily settlement)
- **Finnhub** (free) + **Marketaux** (100 req/day free) — news with sentiment

**Optional one-time:** **Databento** (~$125 free credit, pay-per-use) — a single narrow historical options pull (e.g. SPY/SPX, bounded window) to seed Phase-2 studies. Keep it narrow; OPRA is the highest-volume feed and broad pulls burn credit fast.

**Considered and rejected:**
- **Massive / Polygon** — Polygon rebranded to Massive (Oct 30, 2025); options data now starts ~$99/mo. Strong product, but tastytrade supplies the same metrics free, so unnecessary.
- **Tradier** ($10/mo) — was the earlier plan; redundant once tastytrade is free.
- **TradingView** — no public market-data API (only a Charting Library + Datafeed spec). Not a data source, but the Charting Library is a candidate for the eventual frontend.

---

## 6. Architecture & Hosting

**Host:** Dell OptiPlex 7080, Ubuntu, headless; 100 GB dedicated partition. Compute is light and I/O-bound — **no GPU needed**. Runs as background services without disturbing other use of the box.

**Pipeline:**
1. **Collector** — `systemd` timers:
   - Hourly (market hours): tastytrade chains + market-metrics; yfinance OHLCV/VIX.
   - Daily (EOD): FRED (rates/credit/breakevens); vix_utils (VIX-futures term structure).
   - Periodic: Finnhub/Marketaux news.
2. **Storage** — DuckDB over Parquet, partitioned by date.
3. **Derive** — expected move, own IVR, skew, percentiles, VRP, regime flags.
4. **Present** — v1: terminal / Jupyter against DuckDB (optional `rich`/`textual` TUI). Intermediate: Streamlit on the LAN (view from another machine's browser). Later: web frontend.

---

## 7. Storage Strategy

Storage isn't the constraint — collection **discipline** is (same editorial principle as the display).

- **Engine:** DuckDB + Parquet, date-partitioned. Single-file, no server.
- **What to collect:** the liquid index/ETF complex + a **strike band around spot** (the strikes needed for cone/skew), not full chains.
- **Footprint:** ~**250–300 MB/year** reduced (≈3 GB/year even with full chains for the focused set). → **100 GB = decades of runway.**
- **Seed policy:** store IVR, IVx, IV−HV, and reduced snapshots per underlying **from day one** — kilobytes/day, and the only thing that can't be recreated later.

---

## 8. Tech Stack

- **Core:** Python, `pandas`, `numpy`, `scipy.stats`
- **Data clients:** `tastytrade` (tastyware SDK), `yfinance`, `fredapi`/`pandas-datareader`, `vix_utils`, `httpx`
- **Metrics:** `pandas-ta`
- **Storage:** `duckdb`, Parquet
- **Scheduling:** `systemd` timers
- **Presentation:** Jupyter (v1); optional `rich`/`textual`; `streamlit` (intermediate)
- **Phase-2 research:** `arch` (GARCH), `pyextremes` (EVT/GPD), `quantstats`

---

## 9. Scope

**In scope — v1:** per-underlying vol panel, four-question state strip, expected-move cone, filtered news, fear/greed composite, collector + storage + terminal/Jupyter read-out, background accrual of implied-vol history.

**Deferred — Phase 2 (research / predictive):**
- Full Breeden-Litzenberger risk-neutral density (beyond the cone summary).
- **Conditional base rates** — "when the setup looked like X (term structure, skew, VRP), here's the distribution of forward outcomes." Base rates, not predictions; the intended differentiator. Needs accrued history.
- Tail / risk modeling — EVT (peaks-over-threshold + Generalized Pareto), CVaR / expected shortfall, GARCH, Monte Carlo with fat-tailed innovations, scenario replay (2008, Mar 2020, Aug 2024 yen-carry unwind, Apr 2025 tariff selloff). Note: **no Gaussian parametric VaR** — it understates tail risk.
- tastytrade-style empirical studies — manage-winners-at-50%, 45-DTE sweet spot, strangle vs. straddle, IVR-conditioned returns.
- Percentile-of-swings engine (swing detection → empirical distribution → rank current move). Could be promoted to v1.5.
- Web frontend.

---

## 10. Key Formulas / Definitions (reference)

- **IV Rank:** `(current IV − 52wk low IV) / (52wk high IV − 52wk low IV)`
- **IV Percentile:** `(# of last 252 trading days with IV < current) / 252`
- **IVx:** VIX-style implied vol per expiration cycle (variance-strip method)
- **Expected move (1 SD, to horizon):** `≈ price × IV × sqrt(DTE/365)`; straddle approximation `≈ ATM straddle × ~0.85`. Daily shortcut: `SPX daily expected % ≈ VIX / 16` (√252 ≈ 16)
- **Delta as probability:** `|delta| ≈ P(expire ITM)`; 16Δ ≈ 1 SD, 5Δ ≈ 2 SD; `P(touch) ≈ 2 × P(ITM)`
- **VRP / IV−HV:** implied vol minus realized (historical) vol — the premium harvested by sellers
- **Risk-neutral density (Breeden-Litzenberger):** `f(K) ∝ e^(rT) · ∂²C/∂K²` (smooth the IV surface before differentiating)

---

## 11. Open Decisions for Next-Phase Planning

1. **Point-in-time correctness.** v1 PIT-correct (backtest-survivable) vs. live-snapshot only. PIT is more work but enables the Phase-2 studies. *Decide first — it shapes the collector and schema.*
2. **Underlying universe.** Which liquid single names beyond the index/ETF core.
3. **v1 presentation.** Pure Jupyter vs. `rich`/`textual` TUI vs. early Streamlit.
4. **Historical seed.** Buy the one-time Databento pull now, or wait for self-collected history to accrue.
5. **DuckDB schema.** Concrete table design mapping tastytrade `/market-metrics` + chain fields to what both the v1 panel and Phase-2 studies need. *(Immediate next build step.)*

---

## Phasing summary

- **Phase 1 (now):** Collector + DuckDB + terminal/Jupyter view of the core watchlist (per-underlying vol panels + four-question state strip + cone + news + fear/greed). ~$0/mo. Background accrual of implied history.
- **Phase 2 (later):** Conditional base rates, tail/risk modeling, empirical studies, full RND, web frontend — built on accrued history.
