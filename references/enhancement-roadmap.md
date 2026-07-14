# TradingView MCP — High-Value Roadmap

A prioritized set of additions to make this MCP dramatically more powerful for an AI that *builds and optimizes* trading systems. Ranked by impact ÷ effort. The killer insight from the source: **`indicator_set_inputs` + `pine_smart_compile` + `data_get_strategy_results` already exist as primitives** — they just need to be wired into an autonomous loop. That makes the highest-value item far cheaper than it looks.

Effort key: S = ~½ day, M = 1–3 days, L = 1+ week. Risk reflects fragility of the underlying internal API.

---

## TOP 5 (build these first)

### 1. `strategy_optimize` — parameter sweep / grid optimization loop ⭐
- **What**: One call runs a grid/random sweep over strategy inputs. For each combo: `indicator_set_inputs` → recompile → wait for tester recompute → `getStrategyResults` → record. Returns a ranked table (by Sharpe / net profit / profit factor / a custom objective) plus the best input set. Supports `coarse-then-fine` and early-abort on degenerate combos (0 trades).
- **Why**: This is *the* single biggest force-multiplier. Manual sweeps are the most tedious part of strategy dev; an agent doing 50 combos unattended turns hours of clicking into one tool call. Every other backtest feature feeds off this.
- **Implementation**: Pure orchestration over **existing** primitives — `setInputs()` (indicators.js), `smartCompile()` (pine.js), `getStrategyResults()` (data.js). The one new hard part is a **recompute-settled detector**: poll `getStrategyResults()` until a metric stabilizes across 2 reads, or watch the tester's `reportData()` version/`performance()` promise resolve (the source already accesses `strat.reportData()` / `strat.performance()` — add a "report generation" counter check). Cap combos (e.g. ≤100) and stream progress. No new external deps.
- **Effort**: **M** (primitives exist; the settle-detector and result-table normalization are the real work).
- **Risk**: Medium. Tester recompute timing is the fragile bit — needs a robust settle/timeout to avoid reading stale metrics. Mitigate with a metric-stability poll + max wait.

### 2. `strategy_deep_report` + `trades_export_csv` — normalized metrics & trade export
- **What**: (a) Normalize the raw `reportData()` blob into a stable schema: net profit, profit factor, max DD (abs + %), Sharpe, Sortino, win rate, avg win/loss, expectancy, # trades, longest losing streak. (b) Compute metrics TV omits *from the trade list itself* — Sharpe/Sortino on per-trade returns, R-multiples, MAE/MFE if available. (c) Dump the full trade list to CSV/parquet on disk and return the path.
- **Why**: `getStrategyResults` returns whatever keys TV happens to expose, with inconsistent naming across TV versions — an agent can't reliably reason over it. A stable schema makes objective #1's ranking trustworthy. CSV export unlocks every downstream analysis (Monte Carlo, walk-forward, external notebooks) and is the #1 thing users do manually today (per research: "click the export icon... download as CSV").
- **Implementation**: Extends `getTrades()` (already paginated, capped at 20 — raise the cap for export). Read the full `ordersData()` array via CDP, pair entry/exit rows into round-trip trades, compute derived stats in JS, write file with Node `fs`. Map TV's localized metric labels → canonical keys with a lookup table (version-tolerant). Parquet via a tiny dep (`parquetjs`) or just emit CSV and let a notebook convert.
- **Effort**: **S–M**.
- **Risk**: Low–Medium. Metric-label drift across TV versions; the canonical-key map needs occasional maintenance. Pairing entry/exit rows must handle pyramiding/partial exits.

### 3. `montecarlo_trades` — Monte Carlo + walk-forward on the trade list
- **What**: Given the exported trade list (from #2), run N (e.g. 1000) bootstrap reshuffles/resamples of the trade-return sequence → distribution of final equity, max DD, and CAR/MDD. Return percentiles (p5/p50/p95), probability of ruin, and DD confidence bands. Add a `walk_forward` mode that drives #1 over rolling in-sample/out-of-sample windows (via `chart_set_visible_range` / `replay_start` date windows) and reports IS-vs-OOS degradation.
- **Why**: A single backtest number is a trap — it's one path through history. Monte Carlo on the trade list is the cheapest, most credible robustness check and directly answers "is this curve-fit?" Walk-forward is the gold standard the agent should run before trusting any optimized params from #1.
- **Implementation**: **Pure local computation in JS/Python** — no TV calls needed once trades are exported. Bootstrap = sample-with-replacement over per-trade P&L, accumulate equity, repeat. Walk-forward reuses #1's loop with date-window control already present (`chart_set_visible_range`, `replay` date params). Could ship as a Python sidecar invoked via Bash for numpy speed, or stay in-process.
- **Effort**: **S** for Monte Carlo (offline math), **M** for walk-forward (orchestration + window control).
- **Risk**: Low. It's deterministic offline math; main caveat is methodology correctness (resampling assumes trade independence — document the assumption).

### 4. `chart_vision_verify` — screenshot → multimodal QA / visual regression
- **What**: `capture_screenshot` already returns a file path. This tool feeds that image to a vision model with a structured question ("Does the indicator render? Are labels at plausible price levels? Did the strategy plot trades?") and returns pass/fail + findings. A `visual_regression` mode diffs a Pine change's before/after screenshots and flags unexpected visual changes.
- **Why**: Pine compiles clean but renders wrong constantly (off-by-one offsets, `na` gaps, labels off-screen, wrong scale). Numeric tools (`data_get_pine_lines`, `data_get_study_values`) miss "it looks broken." Vision closes the QA loop so the agent can self-verify a Pine change actually *works visually* — the gap RESEARCH.md calls out as untested. The CLAUDE.md already advises "use a screenshot for visual context."
- **Implementation**: Capture exists. The vision call goes through Claude's own multimodal capability (the agent reads the saved PNG directly — no new service), or an Anthropic API sidecar for a standalone score. Visual regression = pixel/structural diff (e.g. `pixelmatch`) between two captures around the same visible range, with a similarity threshold.
- **Effort**: **S** (the agent can already Read images; formalizing pass/fail + a diff helper is the only code).
- **Risk**: Low. Screenshots are stable; main risk is non-determinism in "looks right" judgments — mitigate by asking structured, falsifiable questions.

### 5. `pine_explain_error` + repaint/lint pass — Pine intelligence layer
- **What**: (a) `pine_explain_error`: maps cryptic pine-facade errors (which `pine_check` already returns) to plain-English cause + concrete fix + corrected snippet, via a curated error→fix knowledge base. (b) A **repaint detector** that statically scans for the classic footguns: `request.security()` without `lookahead=barmerge.lookahead_off` / `[1]` offset, `calc_on_every_tick`, future-leaking historical refs. (c) Auto-input extraction: parse `input.*()` calls → emit the input schema so #1 knows what to sweep.
- **Why**: RESEARCH.md names Pine dev as the *strongest* use case and the compile-fix loop as where agents add most value. `pine_analyze` only catches array/loop bugs today — repaint is the #1 silent killer of "great backtest, dead live." Auto-input-extraction is the connective tissue that lets `strategy_optimize` (#1) discover sweepable params without a human listing them.
- **Implementation**: Extends `analyze()` in pine.js (already an AST-ish static pass). Repaint = regex/AST rules over the source. Error-explainer = lookup table keyed on pine-facade `errors2[].message` substrings (the source already parses these). Input extraction = parse `input.int/float/bool/string/source(...)` with `title`/`defval`/`minval`/`maxval`/`step` → feed the sweep grid.
- **Effort**: **S–M** (extends existing static analyzer; the knowledge base grows over time).
- **Risk**: Low. Offline, no TV dependency. Repaint detection has false positives — report as warnings, not blockers.

---

## STRONG SECOND TIER

### 6. `alert_to_execution` bridge — TradingView alert → webhook → broker MCP
- **What**: Generate the correct strategy-alert JSON template (with `{{strategy.order.action}}`, `{{strategy.order.contracts}}`, `{{strategy.order.price}}`, `{{ticker}}`, `{{strategy.position_size}}`), wire `alert_create` to emit it, run a tiny local webhook receiver, parse the payload, and route to a connected **broker MCP** (`place_order`/`modify_order`) — with a mandatory **human-confirm gate** before any live order.
- **Why**: This is the "build → optimize → *deploy*" closer. If a broker MCP is already connected, the only missing link is the alert→broker translator. Highest *strategic* value, but the live-money risk drops it below the research tools.
- **Implementation**: Template generator is trivial (research confirms exact placeholder syntax). Webhook = local Express/Flask receiver. Map TV JSON → the broker's `place_order` params (symbol/exchange/qty/txn type). **Never auto-fire** — surface for human confirmation (the standard financial-action safety rule: a human confirms every live order).
- **Effort**: **M**.
- **Risk**: **High** (real capital). Needs idempotency keys, position-size sanity caps, kill-switch, and confirm-before-execute. Treat as semi-automated only.

### 7. `tv_selftest` + auto-reconnect / selector-health — reliability layer
- **What**: A one-call diagnostic that exercises each tool category against the live chart (read OHLCV, read a study value, read pine lines, compile a trivial script, read tester) and reports which selectors/internal-API paths still work after a TV update. Add CDP auto-reconnect (re-resolve the page target on `ECONNREFUSED`) and a version-pin readout.
- **Why**: RESEARCH.md's #1 stated limitation: "TradingView Desktop updates can break any tool at any time." A selftest turns a silent breakage into a precise diagnosis ("`reportData()` path changed") and tells the agent which tools to trust — essential before an unattended sweep (#1) runs for an hour.
- **Implementation**: Extends `tv_health_check`/health.js. Each check probes one internal-API path and reports OK/degraded. Auto-reconnect wraps the CDP `evaluate()` in tab.js with target re-resolution + retry. Version detection reads TV's build string from the page.
- **Effort**: **M**.
- **Risk**: Low. Read-only probing; pure resilience win.

### 8. `pine_compile_cache` + library/import manager
- **What**: Local content-hash cache of pine-facade `translate_light` results so re-checking unchanged source is instant; plus helpers to manage `import`/library versions and resolve common library IDs.
- **Why**: During a tight edit-compile loop and especially during sweeps, repeated identical compiles waste seconds and hit pine-facade rate limits. A cache makes #1 and the Pine loop snappier and gentler on the undocumented endpoint.
- **Implementation**: Hash `source` → memoize `check()` result on disk. Import manager parses `import user/lib/ver` lines.
- **Effort**: **S**.
- **Risk**: Low (cache invalidation is the only gotcha — key on full source hash).

---

## THIRD TIER (nice-to-have)

### 9. `data_export_ohlcv` — bars → CSV/parquet
- **What**: Dump `data_get_ohlcv` (already capped at 500) across a date range to disk for offline Python/backtrader/vectorbt research.
- **Why**: Frees research from the context window; lets the agent build models TV can't.
- **Implementation**: Loop `chart_set_visible_range` windows + `getOhlcv`, stitch, write file. **Effort S. Risk Low** (range-stitching dedup).

### 10. `scanner_confluence` — multi-symbol confluence scanner
- **What**: Run a Pine condition / level-confluence check across a watchlist via `batch_run` and return ranked hits (e.g. "RS > 0 AND near PDH AND volume spike").
- **Why**: Turns the existing batch primitive into an actual screener — directly serves the user's confluence-strategy work (per memory).
- **Implementation**: Orchestrate `batch_run` + `data_get_study_values`/`data_get_pine_lines` per symbol, apply a predicate. **Effort M. Risk Low.**

### 11. `correlation_matrix` — cross-watchlist correlation
- **What**: Pull aligned closes across the watchlist, return a correlation/beta matrix.
- **Why**: Portfolio-level reasoning, hedge/divergence spotting (RESEARCH.md §6).
- **Implementation**: `getOhlcv` per symbol → align timestamps → Pearson matrix in JS. **Effort S–M. Risk Low.**

### 12. `chart_snapshot_schedule` — scheduled snapshots / session persistence
- **What**: Periodic `capture_screenshot` + state dump on a cron, persisted to disk; restore a full chart+indicator+input session.
- **Why**: Monitoring and reproducibility; pairs with the platform's scheduled-task tooling.
- **Implementation**: Reuse capture + `chart_get_state`; schedule via external cron/scheduled-tasks MCP. **Effort S–M. Risk Low.**

---

## What I deliberately dropped
- **Fundamentals/news/economic-calendar pull** — better served by dedicated data MCPs; low synergy with this server's CDP strengths.
- **Screener REST integration** — TV's screener API is undocumented and brittle; `scanner_confluence` (#10) over `batch_run` is the lower-risk path to the same outcome.
- **Full autonomous live execution** — out of scope by design; #6 stays semi-automated with a human gate.

## Suggested build order
**#1 → #2 → #3** form one coherent backtest-automation engine (sweep → normalized report/export → robustness). Ship that block first — it's the highest impact and #2/#3 are cheap riders on #1. Then **#5** (Pine intelligence, also cheap, feeds #1's input extraction), then **#4** (vision QA). Reliability (**#7**) before any long unattended sweeps. **#6** (execution bridge) last, gated, once the research loop is trusted.

---

*Grounding sources:* [TradingView strategy alert placeholders](https://www.tradingview.com/support/solutions/43000481368-strategy-alerts/), [TradingView webhook JSON format](https://docs.pickmytrade.trade/docs/tradingview-json-alert-configuration/), [Strategy Tester / Deep Backtesting metrics](https://www.tradingview.com/support/solutions/43000764138-tradingview-strategy-report-how-to-start/), [TradersPost Pine automation](https://docs.traderspost.io/docs/learn/signal-sources/tradingview). Architecture facts (the `indicator_set_inputs`, `getStrategyResults`/`getTrades`/`getEquity`, pine-facade `translate_light`, CDP `evaluate()` primitives) are from the source at `/Volumes/PortableSSD/new pine/tradingview-mcp/src/core/{indicators,data,pine}.js`.
