# House Pine style + real gotchas already hit

A checklist for any code-generator producing Pine Script v6. Every item below is grounded in a set of real production scripts and the bugs they already hit: two trend/confluence strategies, a contra-pivot SuperTrend indicator/strategy pair, and two Gann-cycle (W15) tools ported from an MQL5 EA. It reads like one house's conventions on purpose — steal the parts that fit yours.

---

## 1. File header & version conventions (always)

- [ ] First line is always `//@version=6`. No exceptions — every file is v6.
- [ ] Immediately follow with a boxed banner comment using `// ===...===` rulers (69 `=` chars), a TITLE line in CAPS naming the strategy/indicator AND the build type, then `(Pine Script v6)`.
- [ ] In the header, write a prose spec of WHAT is implemented and — critically — **WHAT is deliberately left out and why** (e.g. confluence header lists OI/options, broker routing, expiry timers as out-of-scope with the reason "cannot be read for a generic symbol in Pine"). The code-generator must explicitly call out scope cuts, never silently drop a spec item.
- [ ] When porting from another platform (e.g. an MQL5 EA), name the exact source EA/version string in the header (e.g. `"MyEA v11 [W15 Auto Slipp]"`) and state the formula in comments before any code.
- [ ] Close the file with a final `// ===...===` ruler and an `// END` line.

## 2. Section structure & comment density

- [ ] Number the major sections in comments: `// 1. INPUTS`, `// 2. INDICATORS`, `// 3. CONFLUENCE BLOCKS`, ... `// 7. DIAGNOSTICS TABLE`, framed with `// ---...---` sub-rulers.
- [ ] High comment density is the house style. Every non-obvious line gets an inline explanation of intent, not just mechanics (e.g. `// buy: cross-up ABOVE the zero line`, `// step-1 target multiplier = 1`).
- [ ] When a line encodes a faithfulness decision to an original spec/EA, say so inline ("atan twice — faithful to EA", "EA pending-order parity", "matches the EA's CopyRates week scan").

## 3. Naming conventions

- [ ] Input-group label variables are prefixed `grp` (`grpDir`, `grpQty`, `grpAdx`, `grpPPS`, `grpOrb`, `grpMan`, `grpLot`, `grpDraw`). Each `input.*` call passes `group = grpX`.
- [ ] Conditions use compact camelCase booleans: `adxRising`, `emaPCO`/`emaNCO`, `rsiBull`/`rsiBear`, `volOK`, `buyTouch`/`sellTouch`, `orbOkBuy`/`orbOkSell`, `tradable`, `okInputs`.
- [ ] Per-side parallelism in names: every long has a mirrored short (`rs21Long`/`rs21Short`, `emaOkBuy`/`emaOkSell`, `doLongFull`/`doShortFull`). The generator should always produce the mirror.
- [ ] Helper functions are prefixed `f_`: `f_yn`, `f_col`, `f_fmt`, `f_qty`, `f_lot`, `f_mult`, `f_step`. `f_yn(b) => b ? "YES" : "no"` and `f_col(b) => b ? lime : red` are a fixed idiom — reuse verbatim.
- [ ] State-machine flags are `var`-declared with a one-line comment each (`var bool longT1 = false // first 50% on`).

## 4. overlay choice & the force_overlay rule (a real gotcha they solved)

- [ ] **Strategies run `overlay = false`** (own pane for oscillators) and then push price-chart visuals up with `force_overlay = true` on EVERY `plot`/`plotshape`/`table.new` that belongs on price (EMAs, ST line, ORB, entry/exit markers, the diagnostics table). Comment: "force_overlay = true, because overlay = false".
- [ ] Oscillator plots that belong in the strategy's own pane get **no** `force_overlay`. One script = one sub-pane.
- [ ] **Indicators run `overlay = true`** and draw directly; filters (ADX/ATR) are reported in the table instead of plotted.
- [ ] Plots that share a 0–100 scale (RSI, ADX) are on by default; plots with their own scale (MACD, RS-21/55, ATR) default OFF via a `show*` bool and are gated `show ? val : na`, with the tooltip "own scale — rescales pane".

## 5. Input patterns to templatize

- [ ] Every filter is independently toggleable with a `use*` bool (`useVol`, `useRS`, `useOrb`, `useAtr`, `useEma`, `useCond9`, `enforceSq`, `useManual`, `useManualLot`) so the user can disable any leg.
- [ ] Mode selectors use `input.string(..., options=[...])` — `signalMode` ("Recent cross"/"Cross bar only"/"State only"), `pyrType`/`tgtMode`/`slMode`/`trlMode` ("Off"/"Points"/"Percent"/"ATR"), `pilotBig` ("SMALL"/"BIG").
- [ ] Tooltips carry the real-world caveat, not restatement of the label — e.g. volume tooltip: "Turn OFF for symbols with no real volume (e.g. some FX/CFD feeds)"; ORB tooltip: "Intraday only."
- [ ] A "Manual levels (testing / parity check)" input group is the house pattern for any ported formula: `useManual` bool + manual override floats that "reproduce the EA's built-in test numbers for verification" (manWH=4857.506, manWL=4600.687, manClose=4702.698).
- [ ] Distance/threshold inputs come as a `*Mode` string + a `*Val` float, converted to price points later ("Points / Percent / ATR" pattern). Percent is `avg * val / 100`; ATR is `val * atrVal`.

## 6. request.security usage (the RS gotcha — auto-handled now)

- [ ] Use the full 5-arg form: `request.security(compSym, timeframe.period, close, barmerge.gaps_off, barmerge.lookahead_off)`. Never omit gaps/lookahead — `lookahead_off` is mandatory (no repaint).
- [ ] **Benchmark == symbol guard (the ZERO-trades bug):** before using a comparative-RS benchmark, test `benchIsSelf = ticker.standard(compSym) == ticker.standard(syminfo.tickerid)`. If the benchmark equals the chart symbol, RS is symbol-vs-itself == 0 forever, so RS-21/RS-55 can never be >0 or <0 and the strategy takes ZERO trades. Set `rsActive = useRS and not benchIsSelf` and make every RS condition pass-through when inactive: `rs21Long = not rsActive or rs21 > 0`.
- [ ] Surface the self-comparison state in the diagnostics table ("5 RS-21 (self-off)" / "off") rather than silently — the user wants to SEE why a leg is skipped.

## 7. No-volume instrument guard (indices — auto-handled now)

- [ ] Never assume `volume` exists. Compute `hasVol = not na(volume) and volume > 0` and make the volume filter pass-through: `volOK = not useVol or not hasVol or volume > volEma`. Comment: "Indices / some FX-CFD feeds report no volume -> auto pass-through so the strategy still works."

## 8. Gann / ported-EA specific gotchas

- [ ] **atan is applied TWICE on purpose.** `angleDeg = atan(atan(range/sum)) * 180/pi`. This is faithful to the EA — do NOT "simplify" it to a single atan. Comment it ("atan twice — faithful to EA") so a future reader doesn't collapse it.
- [ ] **00:00 UTC reference bar.** Levels are anchored to the M15 candle whose `hour(time,"UTC")==0 and minute(time,"UTC")==0`. This means levels ONLY appear on symbols that actually trade at 00:00 UTC (US500/SPX, FX, futures) on an M15 chart. State this requirement in the header ("Run on a symbol that trades 00:00 UTC ... on M15") and in the dashboard status ("Waiting for levels" / "no levels") rather than failing silently.
- [ ] Week high/low is scanned from chart bars with a running `var` max/min reset on `ta.change(time("W")) != 0` — NOT via `request.security` on a weekly TF — "avoids higher-TF security boundary lag" / "matches the EA's CopyRates week scan."
- [ ] SQ filter band is `[sqMin, sqMax]` (defaults 7.0 .. 18.57) and gates `tradable`; show SQ in red/green against the band.
- [ ] Martingale recovery (lot ×2 each step, target multipliers `[1,1,2,3,4...]` via `f_mult`) is drawn/simulated in the cycle indicator but described as "ORDER MANAGEMENT and is not drawn" in the levels-only build — keep these two scopes distinct.

## 9. Defensive math / na-handling (always)

- [ ] Guard division by zero before every ratio: `ratio = compClose != 0 ? close / compClose : na`; gate the whole formula chain behind an `okInputs`/`ok` boolean (`not na(useWH) and not na(useWL) and not na(useC) and useWH > useWL and sm > 0`) and produce `na` downstream when not OK.
- [ ] Lot/quantity normalization is the standard clamp-then-floor-to-step idiom: `math.max(lotMin, math.min(lotMax, math.floor(c / lotStep + 1e-8) * lotStep))` — note the `+ 1e-8` epsilon before floor to avoid float truncation.
- [ ] Heikin-Ashi is computed manually ("to avoid repaint surprises"), with the `var float haOpen` seeded `na(haOpen[1]) ? (open+close)/2 : ...`. Use `syminfo.mintick` as the wick tolerance for "solid candle" tests.
- [ ] Hoist all `ta.*` calls to global scope so they evaluate every bar (v6 consistency rule) — e.g. `rawBuyCross = ta.crossover(...)` is computed globally, then gated by booleans, never called conditionally inside an `if`. There's an explicit comment about this.
- [ ] Use `nz(...)` when reading a possibly-na running value into arithmetic (`nz(strategy.position_avg_price)`, `nz(trend[1], 1)`).
- [ ] SuperTrend trend seed handles the first-bar na: `na(tup[1]) or na(tdown[1]) ? 1 : ...`.

## 10. Order / position handling

- [ ] Strategies set `process_orders_on_close = true` (the "gap rule" — gap-up/down handled @ close), `calc_on_every_tick = false`, `commission_type = strategy.commission.percent`, `commission_value = 0.03`, `slippage = 1`, `initial_capital = 100000`, `default_qty_type = strategy.fixed`.
- [ ] `pyramiding` is set to the max tranches actually used (2 for the 50%+50% confluence; 50 for the contra, then gated by a `numPyr` input with `maxval = 49`).
- [ ] **Staged entries:** explicit `var bool` tranche flags (`longT1`/`longT2`), reset in a single `if flat` block, with `doLongFull`/`doLongHalf`/`doLongAdd` triggers pre-computed (so they can also be plotted) before the `strategy.entry` calls. Comments mark each `strategy.entry(..., comment="L 50%")`.
- [ ] **Staged exits:** two-stage close — `strategy.close("Long", qty_percent = 50, ...)` guarded by a `longPart` flag so the 50% close fires once, then full `strategy.close` on stage 2.
- [ ] For the indicator build of the same logic, a full **manual position/pyramid engine** replaces `strategy.*`: `var` posSize/posDir/avgPrice/lastAdd/addCnt/trailStop, intrabar exits via high/low, `realized`/`nWins`/`nTrades` tracked by hand, `while` loops for multi-step pyramids in one bar, and `canEnter = posDir == 0 and not exitLong and not exitShort` to avoid entering on the same bar as an exit.

## 11. Session / time guards

- [ ] New-day detection: `ta.change(time("D")) != 0`; new-week: `ta.change(time("W")) != 0`; UTC date stamp `year*10000 + month*100 + dayofmonth` compared to a `var prevStamp` for once-per-day cycles.
- [ ] ORB pattern: `var barInDay` counter reset on `newDay`, accumulate `orbHigh`/`orbLow` while `barInDay <= orbBars`, then `orbReady = barInDay > orbBars and not na(orbHigh) and not na(orbLow)`. "Fade-outside" semantics: sell only when `close > orbHigh`, buy only when `close < orbLow`.

## 12. Dashboards via table.new (always present)

- [ ] Every script ends with a diagnostics table built once as `var table dbg/dash = table.new(position.top_right, cols, rows, frame_color=gray, border_color=gray, border_width=1)` and populated only inside `if barstate.islast` (and gated by a `showTbl`/`showPanel` bool where present).
- [ ] On a strategy with `overlay=false`, the table gets `force_overlay = true` so it floats on the price chart.
- [ ] Table layout is a per-condition checklist: column 0 = condition name, columns 1/2 = LONG/BUY vs SHORT/SELL, using `f_yn` text + `f_col` color. Bottom rows show live state (Position, Pyramids, ADX value, Trades/PnL, Status).
- [ ] Status text is human-readable and explains WHY no trade ("Waiting for levels", "SQ out of range", "WON - paused", "max steps", "self-off", "no levels").

## 13. Labels & markers

- [ ] Set `max_labels_count` (500) / `max_lines_count` (50–100) in the `indicator(...)` call when using `label.new`/`line.new`.
- [ ] Persistent labels use `var label` handles, `label.delete(old)` then recreate on `barstate.islast` (Gann levels), placed at `bar_index + 3` with `style_label_left`.
- [ ] Numbers in labels/cells use `format.mintick` or fixed formatters (`f_fmt => "#.###"`, `f_qty => "#.##"`).
- [ ] Entry/exit shapes: `triangleup`/`triangledown` for full signals, `arrowup`/`arrowdown` for pyramids/adds, `xcross` for exits, with `text="50%"/"+50%"/"100%"` size tiers (`size.small` for primary, `size.tiny` for adds/exits).

## 14. Alert wiring

- [ ] Indicators expose `alertcondition(...)` for each event (BUY / SELL / ADD / EXIT) with a clear title and message. (Strategy builds rely on `strategy.*` + comments rather than `alertcondition`.)

---

### Top 5 "already-burned" gotchas the generator must auto-handle without being asked
1. **RS benchmark == chart symbol → ZERO trades** (RS pinned to 0). Detect with `ticker.standard()` and pass-through.
2. **Indices/some FX-CFD have no volume** → `not na(volume) and volume > 0` guard, filter passes through.
3. **Gann uses atan TWICE intentionally** — never simplify.
4. **00:00 UTC M15 reference bar** — Gann levels only appear on symbols trading at 00:00 UTC on M15; say so loudly in header + dashboard status, don't fail silently.
5. **Division-by-zero / na inputs** — gate every formula chain behind an `okInputs` boolean and emit `na`, plus `+1e-8` epsilon before floor-to-lot-step.
