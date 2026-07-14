# TradingView MCP — Complete Capability Map

> **Attribution:** this documents the **TradingView MCP by [@LewisWJackson](https://github.com/LewisWJackson/tradingview-mcp-jackson)** — a separate open-source project that this engine drives. Any `src/…` file/line references point into *that* repo, not this one.

> Scope note: this audit covers the source as it exists in the repo. The server registers exactly **78 MCP tools** (`grep "server.tool(" src/tools/*.js` → 78). The deferred-tool list shown to me also names `morning_brief`, `session_get`, and `session_save`, but those are **not present anywhere in this source tree** (`grep -rln "morning_brief\|session_save\|session_get"` returns nothing) — they belong to a different/newer build and are excluded here. Note also a versioning discrepancy: `package.json` says `version: 1.0.0` while `src/server.js:22` advertises `version: '2.0.0'`, and `CLAUDE.md` claims "68 tools" while the server instructions and README say 78.

---

## 1. Architecture

### Transport: MCP stdio → CDP → TradingView Desktop (Electron)

The server is a standard MCP stdio server. `src/server.js:18-70` constructs an `McpServer` and connects it over `StdioServerTransport` (`src/server.js:93-94`). All 14 tool groups are registered in `src/server.js:73-86`. A compliance notice is printed to **stderr** (not stdout) so it doesn't corrupt the MCP framing (`src/server.js:88-90`).

Every tool ultimately reaches TradingView through the **Chrome DevTools Protocol** against a fixed host/port `localhost:9222` (`src/connection.js:5-6`). TradingView Desktop is an Electron (Chromium) app; launching it with `--remote-debugging-port=9222` exposes the CDP endpoint. The README frames this as the same standard debug interface used by VS Code/Slack/Discord, deliberately opt-in.

**Target discovery** (`src/connection.js:90-97`): the server hits `http://localhost:9222/json/list`, then picks the first `page` target whose URL matches `tradingview.com/chart`, falling back to any `tradingview` page. It connects via `chrome-remote-interface` and enables the `Runtime`, `Page`, and `DOM` domains (`src/connection.js:73-78`).

**Connection resilience** (`src/connection.js:50-88`): `getClient()` caches a single client and does a liveness probe (`Runtime.evaluate('1')`); on failure it reconnects. `connect()` retries up to `MAX_RETRIES=5` with exponential backoff capped at 30s (`src/connection.js:7-8, 83`).

**The core primitive — `evaluate()`** (`src/connection.js:106-125`): nearly every tool injects a JavaScript string into the page via `Runtime.evaluate({ expression, returnByValue: true, awaitPromise })` and gets the value back by-value. `evaluateAsync()` is the `awaitPromise:true` variant used for promise-returning page APIs (e.g. symbol search, layout load). Exceptions in the page surface as thrown JS errors.

### The known internal API paths (the heart of it)

`KNOWN_PATHS` (`src/connection.js:11-27`) hardcodes the undocumented TradingView globals discovered via live probing:

- `window.TradingViewApi._activeChartWidgetWV.value()` — the active chart widget (symbol/resolution/studies/shapes). This is the single most-used path; nearly every chart/data tool starts here.
- `window.TradingViewApi._chartWidgetCollection` — multi-pane layout collection (`pane.js`, `stream.js`).
- `window.TradingView.bottomWidgetBar` — controls the Pine Editor / Strategy Tester bottom panels.
- `window.TradingViewApi._replayApi` — bar-replay engine.
- `window.TradingViewApi._alertService` — alerts (probed in `discover` but the alert tools actually use REST + DOM, see §2).
- `mainSeriesBars` path → `...model().mainSeries().bars()` — the raw OHLCV ring buffer (`valueAt(i)` returns `[time, o, h, l, c, vol]`).
- Strategy data via `model().model().dataSources()` → a source with `.performance()`, `.ordersData()`, `.reportData()`.

`verifyAndReturn()` (`src/connection.js:139-145`) checks a path exists before a tool uses it, returning the path **string** so callers interpolate it into their own injected JS.

### Monaco / React-fiber injection (Pine Editor)

Pine Editor access does **not** go through any public API — it walks the React fiber tree to reach the Monaco editor instance (`src/core/pine.js:9-36`, the `FIND_MONACO` IIFE):

1. `document.querySelector('.monaco-editor.pine-editor-monaco')` finds the container.
2. It climbs up to 20 parent elements looking for a key starting with `__reactFiber$`.
3. From that fiber it walks `.return` up to 15 times looking for `memoizedProps.value.monacoEnv` whose `env.editor.getEditors()` returns a live editor.
4. Returns `{ editor, env }`, giving direct `getValue()`/`setValue()`/`getModel()`/`getModelMarkers()` access.

`ensurePineEditorOpen()` (`src/core/pine.js:42-74`) opens the panel via `bottomWidgetBar.activateScriptEditorTab()`/`showWidget('pine-editor')` (or clicks `[aria-label="Pine"]`), then polls for Monaco up to 50×200ms (~10s).

### pine-facade.tradingview.com endpoints (HTTP, not CDP)

Three categories of operation bypass CDP and call TradingView REST endpoints directly (from Node `fetch` or page-context `fetch`):

- **`translate_light`** (`src/core/pine.js:190`): `POST https://pine-facade.tradingview.com/pine-facade/translate_light?user_name=Guest&pine_id=00000000-...` with `source` form-encoded. This is the server-side Pine compiler. Called from **Node**, so it works with no chart open.
- **Saved-script list/get** (`src/core/pine.js:546, 567, 593`): `GET /pine-facade/list/?filter=saved` and `/pine-facade/get/{id}/{ver}`, called from the **page** with `credentials:'include'` (so they use the logged-in TradingView session cookies). Used by `pine_open` and `pine_list_scripts`.
- **Symbol search** (`src/core/chart.js:225`): `https://symbol-search.tradingview.com/symbol_search/v3/` — public, no auth, called from Node.
- **Alerts list** (`src/core/alerts.js:77`): `https://pricealerts.tradingview.com/list_alerts` from the page with cookies.

### CLI mirror

A parallel CLI (`src/cli/…`, `bin: { tv: ... }`) wraps the same `src/core/*` modules, plus a `stream` command surface (`src/core/stream.js`) that has **no MCP tool** — see §5.

---

## 2. Full tool inventory (78 tools)

Every tool wraps a `core/*` function and returns `{ success, ... }` via `jsonResult()` (`src/tools/_format.js`), which JSON-stringifies with 2-space indent into MCP text content.

### Launch / health (4) — `src/core/health.js`
- **`tv_health_check`** — Connects CDP, reads `window.location`, `document.title`, and chart symbol/resolution/chartType; reports `api_available` (`health.js:8-43`). Gotcha: returns `success:true` with `api_available:false` if the chart object isn't ready yet.
- **`tv_discover`** — Enumerates the six known API globals and lists up to 50/30/20 method names on each (`health.js:45-89`). This is the diagnostic that tells you which internal paths still exist after a TV update.
- **`tv_ui_state`** — DOM-scrapes which panels are open (bottom/right by pixel size), buckets visible buttons into screen regions (`top_bar`, `toolbar`, `left_sidebar`, `pine_header`, `bottom_bar`), and flags "key buttons" (Add to chart / Save and add / errors / unsaved version) via regex (`health.js:91-160`). Powerful for the agent to "see" the Pine compile state without a screenshot.
- **`tv_launch`** — Cross-platform binary auto-detect (hardcoded paths → `which`/`where` → macOS `mdfind`), `pkill`/`taskkill` existing instance unless `kill_existing:false`, then `spawn(detached)` with `--remote-debugging-port`, polling `/json/version` for up to 15s (`health.js:162-251`). Gotcha: **destructive** — kills your running TradingView by default.

### Chart control (8) — `src/core/chart.js`
- **`chart_get_state`** — symbol, resolution, chartType (numeric), and `getAllStudies()` → `[{id,name}]` (`chart.js:17-38`). Call once; reuse the entity IDs.
- **`chart_set_symbol`** — `chart.setSymbol(sym,{})` then `waitForChartReady` (`chart.js:40-53`).
- **`chart_set_timeframe`** — `setResolution` + ready-wait (`chart.js:55-65`).
- **`chart_set_type`** — Maps names→numbers (Bars 0 … HollowCandles 9) and validates 0-9 integer (`chart.js:67-85`). Rejects "10"/"-1"/"1.5".
- **`chart_manage_indicator`** — add: `createStudy(name,false,false,inputArr)`, diffs study IDs before/after to return the new `entity_id` (`chart.js:91-103`). remove: `removeEntity(entity_id)` (requires the ID). Gotcha: requires **full** study names ("Relative Strength Index", not "RSI") — short names silently fail.
- **`chart_get_visible_range`** / **`chart_set_visible_range`** — `getVisibleRange()`/`getVisibleBarsRange()`; set works by scanning bars for matching timestamps then `timeScale().zoomToBarsRange(fromIdx,toIdx)` (`chart.js:118-158`). Gotcha: `from`/`to` are validated finite (`requireFinite`), and "set" returns both `requested` and `actual` ranges since snapping to bar indices is approximate.
- **`chart_scroll_to_date`** — Parses ISO or unix, computes a ±25-bar window from the resolution's seconds-per-bar, then `zoomToBarsRange` (`chart.js:160-197`).
- **`symbol_info`** — `symbolExt()` full metadata (`chart.js:199-212`).
- **`symbol_search`** — Public `symbol_search/v3` REST, strips `<em>` highlight tags, caps at 15 results (`chart.js:214-241`). Works without a chart open.

### Reading chart data (10) — `src/core/data.js`
- **`data_get_ohlcv`** — Reads raw bars from `mainSeries().bars().valueAt()`. `summary:true` returns high/low/range/change%/avg-volume/last-5-bars; otherwise up to 500 bars (`data.js:62-107`). Always use summary.
- **`quote_get`** — Last bar OHLCV + best-effort bid/ask/header-price DOM scrape + symbol ext (`data.js:245-278`). Bid/ask only populate if a DOM panel is on screen.
- **`depth_get`** — Pure DOM scrape of an order-book/DOM panel, classifies bid/ask rows by class/HTML regex, computes spread (`data.js:280-322`). Gotcha: returns an error unless the DOM panel is already open; classification is heuristic.
- **`data_get_study_values`** — Iterates `dataSources()`, reads each study's `dataWindowView().items()` `_title/_value` pairs, skipping the empty sentinel `'∅'` (`data.js:324-358`). This is the canonical "current readings of all indicators" tool.
- **`data_get_indicator`** — `getStudyById(id)` → `isVisible()` + `getInputValues()`, filtering out giant string inputs (>200/>500 chars) (`data.js:109-133`). Gotcha: protected/encrypted indicators return encoded blobs — CLAUDE.md warns to prefer `data_get_study_values`.
- **`data_get_pine_lines`** / **`_labels`** / **`_tables`** / **`_boxes`** — Pine graphics readers; see §5 for the deep mechanism. They dedup, sort, and cap output.
- **`data_get_strategy_results`** / **`data_get_trades`** / **`data_get_equity`** — Strategy/backtest readers; see §2 Strategy below.

### Strategy / backtest (3, in data.js)
- **`data_get_strategy_results`** — Finds the first non-price-study source with `reportData`/`performance`, unwraps `.value()` watched-values, and flattens scalar metrics (`data.js:135-165`).
- **`data_get_trades`** — Same source-find, reads `ordersData()`/`tradesData()`/`_orders`, returns up to 20 trades, keeping only scalar fields (`data.js:167-202`).
- **`data_get_equity`** — Tries `equityData()`, then a `bars()`-style equity series, then falls back to equity-ish keys from `performance()` (`data.js:204-243`). Gotcha: full equity curve is often unavailable; you get a summary object with a `note`.

### Pine development (11) — `src/core/pine.js` (see §3 for the pipeline)
`pine_get_source`, `pine_set_source`, `pine_compile`, `pine_get_errors`, `pine_save`, `pine_get_console`, `pine_smart_compile`, `pine_new`, `pine_open`, `pine_list_scripts`, `pine_analyze`, `pine_check`. (That's 12 — `pine_analyze` + `pine_check` are the offline/server pair.)

Notable gotchas: `pine_new` just `setValue()`s a template (`pine.js:508-535`) — it does **not** click TV's "New" menu, so it overwrites the current editor buffer. `pine_save` dispatches Ctrl+S then hunts a dialog "Save" button for unsaved scripts (`pine.js:347-377`). `pine_open`/`pine_list_scripts` depend on the logged-in cookie session.

### Pine graphics readers (4, in data.js) — see §5.

### Replay (6) — `src/core/replay.js`
- **`replay_start`** — Checks `isReplayAvailable()`, shows toolbar, **awaits** `selectDate(ts).then(...)` inside the page (the comment at `replay.js:27-37` documents that fire-and-forget caused issue #26), then polls up to 30×250ms until `isReplayStarted() && currentDate!=null`. On timeout it calls `stopReplay()` and throws a helpful message.
- **`replay_step`** — `doStep()` then polls up to 12×250ms for `currentDate` to change (`replay.js:59-75`).
- **`replay_autoplay`** — **Validates speed against a fixed whitelist `[100,143,200,300,1000,2000,3000,5000,10000]` BEFORE any CDP call** (`replay.js:6, 77-93`); the comment warns invalid values "corrupt cloud account state permanently."
- **`replay_stop`** — `stopReplay()` if started; idempotent. Tests assert it must **not** call `hideReplayToolbar` (`replay.test.js:280-283`).
- **`replay_trade`** — `buy()`/`sell()`/`closePosition()` then returns `position()` and `realizedPL()` (`replay.js:106-120`). This is **simulated** paper trading inside replay, not real orders.
- **`replay_status`** — Full state incl. position and realized P&L (`replay.js:122-142`).

All replay core functions support dependency injection (`_deps`) for unit testing (`replay.test.js`).

### Batch / multi-symbol (1) — `src/core/batch.js`
- **`batch_run`** — Iterates `symbols × timeframes`, sets each via the chart collection (preferred) or chart API, waits for ready + `delay_ms` (default 2000), then runs one of three actions:
  - `screenshot` → CDP `Page.captureScreenshot`, writes `batch_{symbol}_{tf}_{ts}.png` to `screenshots/` (path separators stripped, `batch.js:38-46`).
  - `get_ohlcv` → uses `chart.exportData(...)` (a different path than `data_get_ohlcv`!), returns bar count + last bar (`batch.js:47-57`).
  - `get_strategy_results` → DOM-scrapes the Strategy Tester panel for `reportItem`/`metric` label/value pairs (`batch.js:58-73`).
  Gotcha: unknown actions return an error per-combo but the overall call still `success:true` with per-iteration results.

### Drawing (5) — `src/core/drawing.js`
- **`draw_shape`** — One point → `createShape`; two points → `createMultipointShape`. Supports `shape` (horizontal_line, vertical_line, trend_line, rectangle, text), `overrides` (JSON style), `text`. Coordinates are `{time: unix, price}` and validated finite. Returns the new `entity_id` by diffing `getAllShapes()` (`drawing.js:10-45`).
- **`draw_list`** — `getAllShapes()` → `[{id,name}]`.
- **`draw_get_properties`** — Dumps `getPoints()`, `getProperties()/properties()`, visible/locked/selectable, plus the shape's available method names (`drawing.js:59-86`) — useful for reverse-engineering the shape API.
- **`draw_remove_one`** — `removeEntity(id)` with before/after verification.
- **`draw_clear`** — `removeAllShapes()`.

### Alerts (3) — `src/core/alerts.js`
- **`alert_create`** — Pure DOM/keyboard automation: clicks `[aria-label="Create Alert"]` (or Alt+A), sets the price via native input setter + input/change events, fills message textarea, clicks "Create" (`alerts.js:6-73`). Fragile — selector-dependent, returns `source:'dom_fallback'`.
- **`alert_list`** — Clean path: `pricealerts.tradingview.com/list_alerts` REST with cookies, returns structured `{alert_id, symbol, type, condition, ...}` (`alerts.js:75-104`).
- **`alert_delete`** — Only `delete_all:true` is partially supported (opens a context menu requiring **manual** confirmation); individual delete throws "not yet supported" (`alerts.js:106-123`).

### UI control (12) — `src/core/ui.js`
- **`ui_click`** — Click by `aria-label`/`data-name`/`text`/`class-contains` (`ui.js:6-29`).
- **`ui_open_panel`** — open/close/toggle for `pine-editor`/`strategy-tester` (via `bottomWidgetBar`) and `watchlist`/`alerts`/`trading` (via sidebar buttons), with real open-state detection by panel pixel size (`ui.js:31-89`).
- **`ui_fullscreen`**, **`ui_keyboard`** (CDP key dispatch w/ modifier bitmask, `ui.js:163-185`), **`ui_type_text`** (`Input.insertText`), **`ui_hover`**, **`ui_scroll`** (mouseWheel at chart center), **`ui_mouse_click`** (raw x/y, left/right/middle, optional double-click), **`ui_find_element`** (returns positions/visibility for text/aria/css matches, capped 20).
- **`ui_evaluate`** — **Arbitrary JS in the page context** (`ui.js:290-293`). The ultimate escape hatch / biggest power-and-risk tool.
- **`layout_list`** / **`layout_switch`** — `getSavedCharts(cb)` / `loadChartFromServer(id)`, with name/substring matching and auto-dismiss of the "unsaved changes" dialog (`ui.js:104-161`). Note: these are saved **layouts**, distinct from `pane_set_layout` grids.

### Tabs / panes / layouts (8)
- **Panes** (`src/core/pane.js`): `pane_list` (symbols + active index per split), `pane_set_layout` (grid codes s/2h/2v/2-1/1-2/3h/3v/4/6/8/… plus aliases like "quad"/"2x2", via `_chartWidgetCollection.setLayout`), `pane_focus` (clicks the pane's `_mainDiv`), `pane_set_symbol` (focus then set on active chart).
- **Tabs** (`src/core/tab.js`): `tab_list` (CDP `/json/list` filtered to chart pages, extracts `chart_id` from URL), `tab_new`/`tab_close` (Cmd/Ctrl+T / +W via CDP key dispatch — platform-aware), `tab_switch` (CDP `/json/activate/{id}`). Gotcha: `tab_close` refuses to close the last tab.

### Watchlist (2) — `src/core/watchlist.js`
- **`watchlist_get`** — DOM scrape of the right panel via `[data-symbol-full]` rows with price/change cells, falling back to text-scan (`watchlist.js:7-63`). Returns a `source` field telling you which scrape strategy succeeded (or `panel_closed`).
- **`watchlist_add`** — Opens the panel, clicks "Add symbol", `Input.insertText(symbol)`, Enter, Escape (`watchlist.js:65-132`). Picks the **first** search result.

### Capture (1) — `src/core/capture.js`
- **`capture_screenshot`** — Default `method:'cdp'` → `Page.captureScreenshot`, optionally clipped to the chart canvas or strategy-tester bounds (`region:'chart'|'strategy_tester'|'full'`), writes a timestamped PNG to `screenshots/` and returns the **file path** (not image bytes). `method:'api'` triggers TV's own `takeScreenshot()` instead.

---

## 3. The Pine error-proofing pipeline (most important section)

There are **four distinct layers** of Pine validation, each catching different classes of problems. They form a cheap→expensive ladder.

### Layer 1 — `pine_analyze` (offline static analysis, zero network/CDP)
`core.analyze({source})` (`src/core/pine.js:78-184`) is pure string analysis. It exactly catches:

1. **Array out-of-bounds on `array.get`/`array.set`** with a literal integer index, when the array's size is statically known. It tracks array sizes from two declaration forms: `x = array.from(a,b,c)` → size = arg count (empty → 0) (`pine.js:94-100`), and `x = array.new_float(N)` / `array.new<T>(N)` → size = N (null if no literal) (`pine.js:102-107`). Then any `array.get(x, idx)`/`array.set(x, idx)` with `idx<0` or `idx>=size` → **error** (`pine.js:110-128`).
2. **Unguarded `.first()`/`.last()` on an array declared with size 0** → **warning** (`pine.js:130-146`). (Skips the literal `array.first()` builtin form.)
3. **`strategy.entry`/`strategy.close` used without any `strategy(` declaration** → **error** with the hint "did you mean indicator()?" (`pine.js:148-165`).
4. **Old Pine version** (`//@version=` < 5, and not v6) → **info** "consider upgrading to v6" (`pine.js:167-176`).

It returns `{success, issue_count, diagnostics:[{line,column,message,severity}], note}`. Verified by 13 unit tests in `tests/pine_analyze.test.js`. **Limits:** purely syntactic/regex — no dynamic indices, no cross-variable reasoning, no type checking. Tool description over-claims "implicit bool casts" and "bad loop bounds" — those checks are **not** implemented in the source.

### Layer 2 — `pine_check` (server-side compile, no chart needed)
`core.check({source})` (`src/core/pine.js:186-243`) POSTs to **pine-facade `translate_light`** from Node (so it works with TradingView entirely closed). It parses the response's `result.errors2[]` and `result.warnings2[]` into `{line, column, end_line, end_column, message}`, plus a top-level `result.error` string. Returns `{compiled, error_count, warning_count, errors, warnings}`. This is the **real TradingView compiler** — it catches everything the IDE catches (unknown functions, type errors, syntax) without touching the editor. E2E/unit tests confirm valid scripts return clean and `this_function_does_not_exist()` returns `errors2` (`pine_analyze.test.js:270-301`). **Limit:** uses `user_name=Guest` + a zero pine_id, so it validates language correctness but not account-scoped library imports.

### Layer 3 — `pine_smart_compile` (inject + button detection + study-added detection)
`core.smartCompile()` (`src/core/pine.js:429-506`) is the "did it actually attach to my chart" check:

1. Ensures the editor is open, then snapshots `getAllStudies().length` **before** (`studiesBefore`).
2. Scans all `<button>`s and clicks, in priority order: "Save and add to chart" → "Add to chart" → "Update on chart" → a `saveButton`-class button; if none found, dispatches **Ctrl+Enter** as fallback (`pine.js:443-470`).
3. Waits 2.5s, then reads **Monaco markers** via `env.editor.getModelMarkers()` (the same source as `pine_get_errors`).
4. Snapshots study count **after** and computes `study_added = after > before` (`pine.js:487-497`).

Returns `{button_clicked, has_errors, errors, study_added}`. The `study_added` boolean is the key signal: it distinguishes "compiled but produced a chart study" from "clicked a button but nothing attached." Gotcha: `study_added` is `null` if either count read fails, and a re-compile of an existing study ("Update on chart") won't increase the count even on success.

### Layer 4 — `pine_get_errors` (Monaco markers)
`core.getErrors()` (`src/core/pine.js:322-345`) reads `env.editor.getModelMarkers({resource: model.uri})` and maps to `{line, column, message, severity}`. These are the live red-squiggle diagnostics in the editor — they reflect the **most recent** in-editor compile/lint state, including warnings the editor surfaces that `translate_light` may format differently. `pine_get_console` (`pine.js:379-427`) complements this by DOM-scraping the console rows for `log.info()` output and compile messages (timestamp + type classification).

### Recommended loop
`pine_analyze` (instant, offline catch of dumb array bugs) → `pine_check` (full server compile, no chart) → `pine_set_source` → `pine_smart_compile` (attach + confirm `study_added`) → `pine_get_errors`/`pine_get_console` (read residual diagnostics/logs). The RESEARCH.md identifies this compile→error→fix loop as the project's strongest use case.

---

## 4. Limitations & fragility

### Undocumented-API dependence — breaks on TV updates
Every chart/data/replay/pane tool hardcodes private paths like `window.TradingViewApi._activeChartWidgetWV.value()._chartWidget.model().model().dataSources()` (`connection.js:11-27`, used throughout). The README's own CAUTION admits these "can change or break without notice." `tv_discover` exists precisely so you can detect which paths died after an update. The deepest, most brittle path is the Pine graphics chain `study._graphics._primitivesCollection.dwglines.get('lines').get(false)._primitivesDataById` (`data.js:11-60`) — five layers of internal field names, any of which can be renamed by minification.

### Monaco React-fiber walk is the single most fragile mechanism
`FIND_MONACO` (`pine.js:9-36`) depends on: the CSS class `.monaco-editor.pine-editor-monaco`, the `__reactFiber$` key prefix, and the shape `memoizedProps.value.monacoEnv.editor.getEditors()`. A React version bump, a class rename, or a prop-shape change silently breaks **all 10 editor-dependent Pine tools** at once. The 200/15/20 loop bounds are heuristic.

### DOM-scraping tools are selector-fragile
`alert_create`, `depth_get`, `watchlist_get/add`, `batch_run get_strategy_results`, `pine_get_console`, and `tv_ui_state` all rely on `[class*="..."]` substring selectors against minified class names. These degrade silently (return empty / `found:false`) rather than erroring loudly.

### Context-size traps
- **`pine_get_source` can return 200KB+** for complex scripts (RESEARCH.md, CLAUDE.md, server instructions all warn). `getSource()` returns the full `editor.getValue()` with no truncation (`pine.js:247-264`).
- **`data_get_indicator` on protected indicators** returns encoded input blobs; it filters strings >200/>500 chars but the structure can still be large (`data.js:125-131`).
- OHLCV is capped at 500 bars, trades at 20, labels at 50/study by default — these caps exist to protect context, but `verbose:true` on the pine readers re-expands payloads with IDs/colors/coordinates.

### Things the tools cannot do
- **No real trading.** `replay_trade` is paper-only inside replay; there is no live-order tool. README explicitly lists "Execute real trades" under "does not do."
- **No reliable individual alert delete** — only `delete_all` (manual confirm) (`alerts.js:106-123`).
- **No full equity curve** in many cases — `data_get_equity` often degrades to summary metrics (`data.js:229-237`).
- **`pine_new` overwrites the editor buffer** rather than opening a fresh TV tab (`pine.js:508-535`) — unsaved work is lost.
- **`pine_open`/`pine_list_scripts`/`alert_list`/`layout_*` require a logged-in session** (cookie-based `credentials:'include'`); they fail or return empty when not authenticated.
- **Single chart target only** — `findChartTarget` picks one page; multi-window setups aren't disambiguated beyond URL match.
- **No audit of `ui_evaluate`** — it runs arbitrary JS in the page with no sandbox; that's by design but is the biggest blast-radius tool.
- The `version` mismatch (1.0.0 in package.json vs 2.0.0 in server) and tool-count drift (68 vs 78) signal documentation is not fully in sync with code.

### Security posture (the one genuinely hardened area)
`safeString()` (`JSON.stringify` wrapping) and `requireFinite()` (`connection.js:36-48`) sanitize all interpolated user input against CDP injection, and `tests/sanitization.test.js` includes a **source-level audit** that fails CI if any `core/*.js` uses manual `.replace(/'/g,...)` escaping or raw `${userInput}` interpolation in an `evaluate()` literal. Filenames are path-separator-stripped (`capture.js`, `batch.js`) to prevent traversal. The `replay_autoplay` whitelist (`replay.js:6,79`) guards against corrupting cloud account state with invalid delay values.

---

## 5. Hidden / underused capabilities

### Pine graphics readers — read custom-indicator drawings as structured data
The four `data_get_pine_*` tools are the standout feature. Custom Pine indicators that draw with `line.new()`, `label.new()`, `table.new()`, `box.new()` produce drawings invisible to normal data tools — these readers extract them by reaching into `study._graphics._primitivesCollection` (`data.js:11-60`, `buildGraphicsJS`):

- **`data_get_pine_lines`** — dedups horizontal levels (`y1===y2`), sorts high→low (`data.js:360-382`). Turns "PDH/PDL/settlement" indicators into a clean price-level list.
- **`data_get_pine_labels`** — text + price pairs, last-50 cap (`data.js:384-402`). Reads "Bias Long ✓", "PDH 24550" annotations.
- **`data_get_pine_tables`** — reconstructs table cells into joined rows by `(tid,row,col)` (`data.js:404-430`). Turns an on-chart analytics dashboard into text rows.
- **`data_get_pine_boxes`** — dedup `{high,low}` zones (`data.js:432-454`).

These let an agent read a third-party indicator's *output* (levels, bias, session stats) without source access or screenshots. **Always pass `study_filter`** (substring match on study name) — and the indicator must be **visible**. `verbose:true` unlocks raw coordinates/colors/styles/IDs.

### `batch_run` actions are richer than they look
Beyond screenshots, `batch_run` can pull OHLCV (via a *different* `exportData` path than the normal OHLCV tool, `batch.js:47-57`) or scrape strategy results across a symbol/timeframe matrix — a one-call multi-symbol scanner that writes per-symbol PNGs to `screenshots/`.

### `replay_trade` — full simulated trading in historical replay
Buy/sell/close with live `position()` and `realizedPL()` feedback (`replay.js:106-120`) turns replay into a backtesting/practice sandbox an agent can drive bar-by-bar (`replay_start` → `replay_step` → `replay_trade` → `replay_status`). Combined with `replay_autoplay`'s speed control, an agent can run a scripted strategy walk-through.

### `stream` — real-time JSONL, CLI-only (no MCP tool)
`src/core/stream.js` (335 lines) is a complete poll-and-dedup streaming engine exposed only through the `tv` CLI, not MCP. It emits JSONL on change for: quote, last-bar, indicator values, pine lines, pine labels, pine tables, and **`streamAllPanes`** — multi-symbol streaming across every pane in the layout (`stream.js:295-335`). RESEARCH.md notes this is intended for human dashboards (piped to monitoring), since real-time streams outpace agent reasoning. A genuinely powerful surface that's invisible to the MCP tool list.

### `draw_shape` with two points + overrides
`point2` unlocks `createMultipointShape` for trend lines and rectangles, and `overrides` accepts arbitrary TV style JSON (`{"linecolor":"#ff0000","linewidth":2}`) (`drawing.js:22-38`). An agent can annotate exact `{time,price}` coordinates derived from the pine-line readers — e.g., draw a box around a zone it just read from `data_get_pine_boxes`.

### `tv_ui_state` as a "screenshot replacement" for Pine state
It surfaces `key_buttons` (Add to chart / Save and add / Update on chart / compile errors / unsaved version) and panel-open booleans (`health.js:129-145`) — letting an agent reason about Pine compile/save state textually (cheaply) instead of capturing and interpreting a screenshot.

### `draw_get_properties` method enumeration
It dumps the shape object's available **method names** (`drawing.js:69`), effectively a live API explorer for reverse-engineering TradingView's shape interface — paired with `tv_discover`, these two are the project's self-documentation tools for surviving TV updates.

### `ui_evaluate` — arbitrary page JS
The unrestricted escape hatch (`ui.js:290-293`): anything not covered by a dedicated tool can be done by injecting raw JS against the live `TradingViewApi`. Powerful for one-off internal-API calls the 78 tools don't wrap.

---

### Key file:line references
- CDP connect + known paths + sanitizers: `src/connection.js:11-27, 36-48, 64-97, 106-125`
- Monaco fiber injection: `src/core/pine.js:9-36`
- Pine pipeline: `analyze` `pine.js:78-184`, `check` `pine.js:186-243`, `smartCompile` `pine.js:429-506`, `getErrors` `pine.js:322-345`
- Pine graphics extraction: `src/core/data.js:11-60` (+ readers `360-454`)
- Strategy/backtest readers: `src/core/data.js:135-243`
- Replay (date-await fix, speed whitelist): `src/core/replay.js:6, 27-37, 77-93`
- Streaming (CLI-only): `src/core/stream.js`
- Tool registration (78 tools, 14 groups): `src/server.js:73-86`
- Security audit tests: `tests/sanitization.test.js:287-328`
