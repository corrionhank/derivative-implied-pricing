# Open Decisions

Five architectural decisions to resolve before building. Listed in dependency order — #1 blocks the schema and collector design; resolve it first.

---

## 1. Point-in-time correctness

**Question:** Does the collector write a fresh record for every run (append-only, `collected_at` timestamps preserved), or overwrite the latest snapshot in-place?

**Implications:**
- **PIT-correct (append):** Every record carries `collected_at`. You can reconstruct "what did the system know at any past time T?" This is required for Phase-2 conditional base rates and any realistic backtest. Costs nothing extra at collection time — same data, just never overwrite.
- **Live-snapshot only (overwrite):** Simpler to implement; the table always holds the current state. But history is gone. Phase-2 studies become impossible unless you switch later (and you can't recreate lost history).

**Recommendation:** PIT-correct from day one. The append design costs nothing at this data volume. The schema in `docs/schema.md` is drafted PIT-correct. The only added complexity is using upsert logic (INSERT OR REPLACE by `collected_at` + `symbol`) to handle collector retries.

**Decision:** _TBD_

---

## 2. Underlying universe

**Question:** Which liquid single names to track beyond the index/ETF core?

**Core (settled):** SPY, QQQ, IWM, /ES, /VX (VIX futures), VIX cash

**Candidates for single names:**
- High-liquidity, options-active names: AAPL, TSLA, NVDA, AMZN, META, GLD, TLT, XLE
- Principle: "track the liquid complex, not single-name noise" — include names where the options market is genuinely informative and liquid

**Implications:** Each added name increases collector time and storage linearly. Given the focused universe aim, a short list (4–8 names) is reasonable. The `symbols.toml` has a placeholder `single_names = []` to fill in.

**Decision:** _TBD — populate `config/symbols.toml` `[single_names]` section_

---

## 3. v1 presentation layer

**Question:** What is the v1 UI?

**Options:**
- **Pure Jupyter** — minimal work; query DuckDB in notebooks and render with matplotlib/plotly. No live refresh. Good for exploration.
- **`rich`/`textual` TUI** — terminal table with live refresh. No browser needed. Fits a headless server well. More work than Jupyter.
- **Early Streamlit** — LAN-accessible from any device. Closest to the final vision. Requires a running process. Intermediate complexity.

**Recommendation:** Start with Jupyter for the first few weeks (data validation, formula check) and promote to Streamlit once the data layer is stable. Skip `textual` unless you want a dedicated watchlist terminal without a browser.

**Decision:** _TBD_

---

## 4. Historical seed (Databento)

**Question:** Buy the one-time Databento pull now (~$125 free credit for a bounded OPRA window), or wait for self-collected history to accrue?

**Implications:**
- **Buy now:** Seeds Phase-2 studies with years of historical options data immediately. OPRA feeds are expensive in bulk — use credit for SPY/SPX only, bounded to a specific window (e.g. 2020–2024 for the stress events listed in the brief). Narrow scope is critical — broad pulls burn credit fast.
- **Wait:** Self-collected data accrues at kilobytes/day. Phase-2 studies depend on history that can't be recreated, but a few months of self-collection is enough to validate the pipeline before spending.

**Recommendation:** Wait until the pipeline is validated (collector running cleanly for 2–4 weeks), then spend the Databento credit on a narrow SPY/SPX window covering the key stress episodes (Aug 2024 yen unwind, Apr 2025 tariff selloff).

**Decision:** _TBD_

---

## 5. DuckDB schema (immediate next build step)

**Question:** Finalize the concrete table design before writing the first collector.

**Draft:** `docs/schema.md` has the full DDL. Key review points:

1. Are all the tastytrade `/market-metrics` fields mapped correctly? (Verify field names against the SDK's `MarketMetrics` model before writing the collector.)
2. Is the option chain strike band (±15% from spot) too wide or too narrow for the skew and expected-move calculations you care about?
3. Does the `derived_metrics` table cover the v1 display requirements from the brief (IV panel, four-question state strip)?
4. FRED series list in `config/symbols.toml` — complete?

**Decision:** _Review schema.md; adjust and mark finalized before starting collector._
