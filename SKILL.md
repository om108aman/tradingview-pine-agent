---
name: tradingview-pine-agent
description: >-
  Zero-error Pine Script automation for TradingView via the local TradingView MCP
  (CDP bridge). Use whenever the user wants to write, build, port, fix, compile,
  validate, backtest, or iterate on a Pine Script v6 indicator, strategy, or
  library in TradingView — or asks to add/remove indicators, read chart/strategy
  data, run a parameter sweep, wire alerts/webhooks, or capture the chart. Drives
  the loop so code compiles on the FIRST chart push with no compile OR runtime
  errors, using pine_check (server-side compiler) + pine_analyze (static) before
  ever touching the chart. Triggers on "pine script", "indicator", "strategy",
  "tradingview", "compile this", "fix this pine error", "backtest", "convert to v6".
---

# TradingView Pine Automation (Zero-Error Loop)

You drive a **live TradingView Desktop** chart through the `tradingview` MCP (78 tools,
Chrome DevTools Protocol bridge on port 9222). Your job: produce Pine Script v6 that
**compiles on the first chart push and runs without runtime errors**, then verify it on
the chart and (for strategies) read the backtest.

This skill supersedes the MCP's bundled `pine-develop` skill, which routes through old
CLI scripts and never uses `pine_check`/`pine_analyze`. **Always use the loop below.**

## The Golden Rule

> Never say "done" until: `pine_check` shows `error_count: 0` **AND** the on-chart
> `pine_smart_compile` shows no error markers **AND** `pine_get_console` is clean of
> runtime errors **AND** (for visuals/strategies) a screenshot or `data_get_*` read
> confirms it actually renders/trades.

Compile-clean ≠ runtime-clean. `pine_check` validates *compilation* only. Runtime traps
(`str.format` brace mismatch, array out-of-bounds on a live bar, `na` propagation,
`max_bars_back`, too-many-securities at runtime) pass `pine_check` and only surface on
the chart — which is why the loop has **two** validation layers.

---

## The Zero-Error Loop

Run these in order. Steps 1–2 are **offline** (no chart needed, fast, free); do them
*before* ever injecting code. Steps 3–6 confirm on the real chart.

### 0. Connect (once)
- `tv_health_check` → confirm CDP is up and a chart is loaded. If it fails, `tv_launch`
  (⚠️ kills the running TradingView by default — warn the user first).
- `chart_get_state` → current symbol/timeframe + entity IDs (call once, reuse).

### 1. Static analysis — `pine_analyze` (offline)
Catches array out-of-bounds, unguarded `array.first()/last()`, bad loop bounds,
`strategy.*` calls with no `strategy()` decl, implicit bool casts. Fix everything it
flags. Zero TradingView round-trip.

### 2. Server-side compile — `pine_check` (offline, AUTHORITATIVE)
POSTs the source to TradingView's real `translate_light` compiler and returns
`errors2`/`warnings2` with **line + column**. This is the workhorse — iterate here, not
on the chart:
- `error_count > 0` → read each `{line, column, message}`, fix the source, re-run
  `pine_check`. Repeat until `error_count: 0`.
- Resolve warnings too (e.g. "function call inside conditional might not execute every
  bar" → hoist the `ta.*`/`request.*` call to global scope).
- **Mentally simulate runtime** for anything `pine_check` can't see — especially
  `str.format` (literal `{`/`}` must be `{{`/`}}`, or build JSON by concatenation),
  array indexing on live bars, and `request.security` repaint. See
  `references/pine-v6-gotchas.md`.

### 3. Inject + compile on chart — `pine_set_source` → `pine_smart_compile`
- `pine_set_source` injects into the Monaco editor.
- `pine_smart_compile` clicks the right "Add/Update on chart" button, reads Monaco error
  markers, and reports `study_added` (true = it actually attached to the chart).
- If `has_errors` → `pine_get_errors` for the markers, fix, re-inject. (These should be
  rare if step 2 was clean — a mismatch usually means a runtime-only issue.)

### 4. Runtime check — `pine_get_console`
Reads the Pine console: runtime errors, `runtime.error()`, and your `log.info()` output.
**A script can compile and still throw here** (this is where `str.format` and array
runtime errors appear). Must be clean.

### 5. Visual / data verification
- `capture_screenshot` (region `chart`) → confirm it renders (lines on screen, labels at
  plausible prices, no `na` gaps). `pine_check` cannot see "it looks broken."
- Custom drawings: `data_get_pine_lines` / `data_get_pine_labels` / `data_get_pine_tables`
  / `data_get_pine_boxes` (always pass `study_filter` = your script's name) to read back
  the actual levels/labels/zones the script drew.
- Plotted values: `data_get_study_values`.

### 6. Strategy backtest (strategies only)
- `capture_screenshot` region `strategy_tester`, then `data_get_strategy_results`
  (net profit, profit factor, max DD, win rate, # trades).
- `data_get_trades` (trade list), `data_get_equity` (equity curve).
- **0 trades is a red flag** — check the house gotchas first (RS-benchmark==symbol,
  no-volume instrument, a date filter, an over-strict confluence). See
  `references/house-style.md`.

### 7. Save
`pine_save` writes to the user's TradingView cloud (handles the name dialog for new
scripts). Only after the user confirms the result.

---

## Pick the right starting template

Seven **compiler-verified** v6 templates ship in `templates/` (each passed `pine_check` with
`error_count: 0`). Start from the closest one instead of from scratch — they already
encode the gotcha fixes:

| File | Use when you need… |
|------|--------------------|
| `templates/indicator-overlay.pine` | Overlay indicator: EMA cross, dynamic coloring, cross markers, `alertcondition` + dynamic `alert()`, dashboard table |
| `templates/indicator-oscillator.pine` | Separate-pane oscillator: normalized 0–100, hline guides, banded `fill`, gradient plot, extreme `bgcolor`, OB/OS alerts |
| `templates/strategy-clean.pine` | Strategy skeleton: % equity sizing, commission/slippage, ATR stop+target via `strategy.exit`, date-range filter, webhook-ready `alert_message` |
| `templates/strategy-webhook-alerts.pine` | Broker automation: one `alert()` firing a **valid JSON payload** (built by concatenation) on confirmed fills + the `{{strategy.order.alert_message}}` alternative |
| `templates/mtf-nonrepaint.pine` | Non-repainting multi-timeframe: `f_secure()` HTF helper with the why-it-doesn't-repaint explanation |
| `templates/dashboard-table.pine` | Polished analytics dashboard: themed `table.new`, position/size input→constant mapping, `var` + `barstate.islast` |
| `templates/session-filter.pine` | Session/time windows: `input.session`, exchange-timezone-aware `time()`, date-range via `timestamp()`, na-safe |

To use one: `Read` the asset → adapt to the request → run the loop. Keep the house style
(below) when adapting.

---

## Top runtime/compile traps (the short list)

Full catalog with WRONG→RIGHT snippets in **`references/pine-v6-gotchas.md`**. The ones
that bite most:

1. **`str.format` literal braces** — a single `{` that isn't a `{0}` placeholder throws a
   *runtime* error that compiles clean. Double them (`{{`/`}}`) or build strings by
   concatenation (preferred for JSON).
2. **`bool` can't be `na`** in v6; `na()`/`nz()` reject bools. Default bools to `false`.
3. **No implicit int→bool**: `if count` → `if count > 0`.
4. **`switch`/`if` returning a unique type** (a `plot.style_*`, `position.*`, color)
   needs a `=>` default / `else`, else it returns `na` → "Cannot call with na".
5. **Lazy `and`/`or`**: a `ta.*`/`request.*` in a short-circuited branch stops running
   every bar → wrong series + a warning. **Hoist all `ta.*`/`request.*` to global scope.**
6. **`transp=` removed** → bake into `color.new(col, transp)`.
7. **Qualifier mismatch**: `series` passed where `simple`/`const` is required (e.g.
   `ta.ema` length, `hline` price, `max_*_count`). Use `input.*` (counts as simple/input).
8. **`plot`/`hline`/`bgcolor`/`fill` must be global** — plot conditionally with a value
   (`plot(cond ? x : na)`), never wrap the call in an `if`.
9. **Repaint**: non-repainting HTF = `request.security(sym, tf, expr[1], lookahead=barmerge.lookahead_on)`
   (confirmed-bar). The `mtf-nonrepaint.pine` template uses the equally-safe
   `expr[1]` + `lookahead_off` variant (one-HTF-bar lag, zero hindsight). Never use bare
   `lookahead_on` without an offset — that leaks the future.
10. **Object/security caps**: ≤500 lines/labels/boxes (set `max_*_count`, reuse/delete
    objects); ≤40 unique `request.*()` (consolidate with tuple returns).

---

## House style (battle-tested Pine v6 conventions)

These conventions come from a set of production Pine v6 scripts and the bugs they already
hit — captured so a generator reproduces them by default. Full checklist in
**`references/house-style.md`**. Highlights:

- `//@version=6` on line 1, then a boxed `// ===…===` banner naming the script + build
  type, a prose spec of **what's implemented and what's deliberately left out and why**.
- Numbered comment sections (`// 1. INPUTS`, `// 2. INDICATORS`…), high comment density,
  inline notes on every faithfulness/scope decision.
- Naming: input-group labels prefixed `grp*`; helpers prefixed `f_`; per-side mirrored
  booleans (`*Long`/`*Short`, `*Buy`/`*Sell`); state flags `var`-declared with a comment.
- Every filter independently toggleable with a `use*` bool; mode selectors via
  `input.string(options=[…])`; tooltips carry the real caveat, not a restatement.
- A `table.new` diagnostics dashboard at `position.top_right`, built `var`, populated only
  in `if barstate.islast`, status text explaining **why** there's no trade.
- **Auto-handle these without being asked** (already-burned gotchas):
  1. RS benchmark == chart symbol → RS pinned to 0 → **zero trades**. Detect with
     `ticker.standard()` and pass-through.
  2. Indices / some FX-CFD feeds have no volume → `not na(volume) and volume > 0` guard,
     filter passes through.
  3. Strategies: `overlay=false` + `force_overlay=true` on price-chart visuals.
  4. Gate every formula chain behind an `okInputs` bool, emit `na` when not OK; `+1e-8`
     epsilon before floor-to-lot-step.
  5. Hoist all `ta.*` to global scope.

---

## Power workflows (beyond a single script)

- **Parameter sweep / optimization**: `indicator_set_inputs` → `pine_smart_compile` →
  wait for tester recompute → `data_get_strategy_results`, looped over input combos, rank
  by your objective. (These primitives already exist; this is the #1 enhancement to
  formalize — see `references/enhancement-roadmap.md`.)
- **Multi-symbol scan**: `batch_run` with `symbols`/`timeframes` for screenshots or OHLCV
  across a watchlist.
- **Replay practice / event study**: `replay_start` (date) → `replay_step` →
  `replay_trade` → `replay_status`.
- **Alert → broker bridge**: generate the `{{strategy.order.*}}` alert template, fire
  webhook JSON via `alert()`. If a broker MCP is connected, route fills there for
  semi-automated execution — **always with a human confirm gate before any live order**.

---

## Reference index

- **`references/pine-v6-gotchas.md`** — the zero-error knowledge base: v5→v6 breaking
  changes, every common compile error → cause → fix, type qualifiers, repainting,
  strategy timing, object limits, plotting rules, performance caps. Read this when fixing
  a stubborn error or writing anything non-trivial.
- **`references/house-style.md`** — the Pine v6 conventions + the 5 already-burned
  gotchas to auto-handle. Read before generating anything non-trivial.
- **`references/mcp-tool-map.md`** — the full 78-tool inventory, the CDP/Monaco
  architecture, the error-proofing pipeline internals, and limitations. Read when you need
  a tool you don't know or are debugging the MCP itself.
- **`references/enhancement-roadmap.md`** — prioritized additions to make the MCP more
  powerful (strategy_optimize, deep report + CSV export, Monte Carlo, vision QA, Pine
  linter, alert→execution bridge). Read when the user asks "what can we add."

## Cautions
- The MCP rides **undocumented** TradingView internals; a TV update can break a selector.
  If a tool returns empty/odd, `tv_discover` shows which internal API paths still exist.
- `pine_get_source` can return 200KB+ on big scripts — avoid unless you must edit; prefer
  keeping the source you authored.
- Pine graphics readers (`data_get_pine_*`) only work when the indicator is **visible**.
- Chart mutations execute immediately. Confirm before destructive acts (`tv_launch`
  kills TV; `draw_clear` wipes drawings) and never auto-fire live broker orders.
