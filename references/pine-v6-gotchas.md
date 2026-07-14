# Pine Script v6 — Zero-Error Knowledge Base

A field manual for writing Pine v6 that compiles on the first try. Every section pairs the **WRONG** pattern that triggers a real error with the **RIGHT** fix. Current to Pine v6 (2024–2026).

> **Top-of-file rule:** every script starts with `//@version=6` on line 1. A missing/wrong version annotation silently downgrades the dialect and makes half the errors below "lie" about their cause.

---

## 1. v5 → v6 Breaking Changes (the silent breakers)

### 1.1 `bool` can no longer be `na`
Booleans are now strictly `true`/`false`. Undefined conditional branches return `false`, not `na`. `na()`, `nz()`, `fixnan()` **reject** bool arguments.

```pine
// WRONG (v5 habit) — won't compile in v6
bool b = na                       // bool can't be na
x = na(myBoolCondition)           // na() doesn't accept bool
crossed = ta.crossover(a, b)
state = na(crossed) ? false : crossed   // na() on a bool

// RIGHT
bool b = false                    // give it a real default
x = na(myFloatValue)              // only use na() on int/float/series
crossed = ta.crossover(a, b)      // already strictly true/false
```

If you genuinely need a tri-state, model it with an `int` (0/1/2) or a separate "isset" flag.

### 1.2 No implicit int/float → bool casting
`if myCount` and `if bar_index` no longer work.

```pine
// WRONG
if bar_index            // int is not a bool in v6
    label.new(...)

// RIGHT
if bar_index > 0        // explicit comparison
    label.new(...)
// or
if bool(bar_index)      // explicit cast
    label.new(...)
```

### 1.3 `switch`/`if` returning na on unique types
Expressions that yield a *unique* type (e.g. `plot.style_*`, `label.style_*`, `color` for unique-typed params) must guarantee a non-`na` result — finish every `switch` with a `=>` default and every `if` with an `else`.

```pine
// WRONG — falls through to na on the unrecognized case
mystyle = switch userChoice
    "line" => plot.style_line
    "hist" => plot.style_histogram
// no default -> returns na -> "Cannot call plot with na style"

// RIGHT
mystyle = switch userChoice
    "line" => plot.style_line
    "hist" => plot.style_histogram
    => plot.style_line          // default branch guarantees non-na
```

### 1.4 Integer division now returns a float (truncation change)
Const-integer division kept the remainder in v6.

```pine
// v5: 5 / 2 == 2     |     v6: 5 / 2 == 2.5
half = 5 / 2          // now 2.5

// RIGHT (if you wanted integer/floor behavior)
half = int(5 / 2)     // 2  — int() truncates toward zero
// or use math.floor / math.round explicitly for intent
idx  = int(len / 2)
```
Note `int()` **truncates toward zero** (not floor); `-2.7` → `-2`. Use `math.floor()` if you need true floor.

### 1.5 `request.*()` is dynamic by default — drop `dynamic_requests=true`
v6 allows `series` symbol/timeframe and `request.*()` inside loops/conditionals automatically. Remove the explicit parameter; the compiler auto-disables dynamics when unneeded.

```pine
// WRONG (v5 leftover) — parameter no longer needed / can confuse intent
indicator("X", dynamic_requests = true)

// RIGHT
indicator("X")
// now legal in v6:
sym = condition ? "AAPL" : "MSFT"
px  = request.security(sym, "D", close)   // series symbol OK
```

### 1.6 Lazy `and`/`or` (short-circuit) — side effects can vanish
v6 stops evaluating once the result is known. A `ta.*` call buried in the dead branch of an `and`/`or` may not run every bar, corrupting its internal state.

```pine
// WRONG — ta.rsi only evaluated when cond is true -> its series breaks
sig = cond and ta.rsi(close, 14) > 70

// RIGHT — hoist stateful calls to global scope
r = ta.rsi(close, 14)         // runs every bar
sig = cond and r > 70
```

### 1.7 `color.*` constant values & defaults changed
`color.red` `#FF5252`→`#F23645`, `color.teal`→`#089981`, `color.yellow`→`#FDD835`; default **label text color is now `color.white`** (was black). Visual-only, but breaks pixel-exact comparisons and makes light-background labels unreadable — set text color explicitly.

### 1.8 `transp` parameter removed everywhere
```pine
// WRONG
plot(close, color = color.red, transp = 70)
bgcolor(color.green, transp = 90)

// RIGHT — bake transparency into the color
plot(close, color = color.new(color.red, 70))
bgcolor(color.new(color.green, 90))
```

### 1.9 History-referencing operator `[]` restrictions
- No history on literals/constants: `6[1]`, `color.red[3]` are illegal.
- UDT field history: reference the **object's** history, then read the field.
```pine
// WRONG
prev = myObj.field[10]

// RIGHT
prev = (myObj[10]).field
```

### 1.10 `timeframe.period` always carries a multiplier
Daily now returns `"1D"`, not `"D"`.
```pine
// WRONG
isDaily = timeframe.period == "D"
// RIGHT
isDaily = timeframe.period == "1D"
// safer: use timeframe.in_seconds() or timeframe.isdaily
isDaily = timeframe.isdaily
```

### 1.11 Strategy migration breakers
- **`when=` removed** from `strategy.entry/exit/order` → wrap in `if`.
- **Default `margin_long`/`margin_short` = 100** (was 0) → unexpected margin calls; set to `0` to match old behavior if needed.
- **`strategy.exit()` now evaluates both** absolute (`limit`/`stop`) and relative (`profit`/`loss`) params, using whichever triggers first.
```pine
// WRONG (v5)
strategy.entry("L", strategy.long, when = longCond)
// RIGHT (v6)
if longCond
    strategy.entry("L", strategy.long)
```

### 1.12 Other v6 traps
- **Mutable vars now correctly typed `series`** — code that fed a reassigned var into a `simple`-demanding param (e.g. `request.security` historically, `ta.ema` length) now errors honestly. Wrap in a function (see §2.5) or restructure.
- **`for` loop bounds re-evaluated every iteration** — hoist a state-mutating boundary into a variable before the loop.
- **`offset` must be `simple int`** (no series).
- **`linewidth` minimum is 1.**
- **Duplicate function parameters** now a hard error (was a warning).
- **Array negative indexing** is now supported (`arr.get(-1)` = last) — convenient, but a leftover manual `size-1` plus a negative index can double-offset.
- **`varip`** unchanged semantically: persists *across* real-time ticks within a bar (doesn't roll back on each tick like `var`); use only for tick-counting/real-time accumulation. Backtests on history treat `varip` like `var`.

---

## 2. Top Compile Errors → Real Cause → Fix

### 2.1 `Mismatched input '...' expecting '...'`
**Cause:** parser hit something unexpected — almost always a problem on the **line above** the highlighted one (missing `)`, trailing comma, premature line end) or bad continuation-line indentation. When it says *"expecting 'end of line without line continuation'"* it's a wrapping/indentation issue.
```pine
// WRONG — comment on a wrapped line kills continuation
x = ta.sma(close, 20) +  // moving average
    ta.sma(close, 50)

// WRONG — continuation line indented a multiple of 4 (looks like a new block)
y = a +
    b               // 4 spaces -> ambiguous

// RIGHT — no comments mid-wrap; continuation indent NOT a multiple of 4
x = ta.sma(close, 20) +
  ta.sma(close, 50)     // 2 spaces is fine for a continuation
```
Rules: 4 spaces per real indent level, **no tabs**, never mix tabs/spaces, no comments inside a wrapped expression, continuation lines use a non-multiple-of-4 indent.

### 2.2 `Could not find function or function reference '...'`
**Cause:** wrong name or capitalization, or calling a strategy function from an indicator.
```pine
// WRONG
s = sma(close, 20)           // v4 name
v = security(...)            // v4 name
// RIGHT
s = ta.sma(close, 20)
v = request.security(...)
```
Use Ctrl+Space autocomplete to confirm exact namespaced names (`ta.`, `request.`, `str.`, `math.`, `array.`, `matrix.`). Strategy calls in an `indicator()` script throw a related "can't use strategy functions" error.

### 2.3 `Undeclared identifier '...'`
**Cause:** variable used before/outside its declaration scope (e.g. declared inside an `if` block, used globally), or a typo.
```pine
// WRONG — sig only exists inside the if-block
if cond
    sig = close > open
plot(sig ? 1 : 0)            // sig undeclared here

// RIGHT — declare in the scope you read it
sig = false
if cond
    sig := close > open
plot(sig ? 1 : 0)
```

### 2.4 `Cannot call 'X' with argument 'Y'='Z' ... type series, expecting simple/const`
**Cause:** a series value passed where a `simple`/`const` is required (qualifier mismatch — see §3).
```pine
// WRONG — length is series (changes per bar)
len = int(close)            // series int
e = ta.ema(close, len)      // ta.ema length wants simple int

// RIGHT — make length simple (from input or const)
len = input.int(20)        // input qualifier (accepted as simple)
e = ta.ema(close, len)
```
For genuinely dynamic lengths, use a function that accepts `series` length (e.g. compute manually, or `ta.ema` works with series length in v6 for many cases — but `max_bars_back` may be needed).

### 2.5 `Cannot use a mutable variable as an argument...` (security)
**Cause:** a `:=`-reassigned variable passed directly into `request.security()`.
```pine
// WRONG
s = 0.0
s := nz(s[1]) + close
t = request.security(syminfo.tickerid, "D", s)

// RIGHT — wrap mutation in a function, pass the call
calcS() =>
    var float s = 0.0
    s := nz(s[1]) + close
    s
t = request.security(syminfo.tickerid, "D", calcS())
```

### 2.6 `... end of line without line continuation`
**Cause:** a line ends mid-expression with no valid continuation, or a wrapped line is mis-indented, or a comment sits inside a wrap. Same fix family as §2.1.

### 2.7 `The function 'X' should be called on each calculation for consistency...`
**Cause:** a function that maintains internal state (`ta.*`, `request.*`, `barssince`, etc.) is called inside an `if`/`for`/ternary so it doesn't run every bar.
```pine
// WRONG
v = cond ? ta.rsi(close, 14) : na
// RIGHT
r = ta.rsi(close, 14)        // every bar
v = cond ? r : na
```

### 2.8 `Script requesting too many securities` / `too many unique request.*() calls`
**Cause:** more than **40 unique** `request.*()` calls (64 on Professional). A `request.security` stored in a var and reused counts **once**; a `request.security` inside a function counts **per call site/iteration**.
```pine
// WRONG — 3 separate calls
o = request.security(sym, "D", open)
h = request.security(sym, "D", high)
c = request.security(sym, "D", close)

// RIGHT — one call, tuple return
[o, h, c] = request.security(sym, "D", [open, high, close])
```
For >127 values use a UDT as the expression. Remove redundant calls; consolidate by symbol+timeframe context.

### 2.9 `Memory limits exceeded` / `Script requesting too many bars`
**Cause:** unbounded history references, huge `max_bars_back`, or array/matrix growth without pruning.
```pine
// WRONG — array grows forever
var prices = array.new_float()
array.push(prices, close)

// RIGHT — cap the size
var prices = array.new_float()
array.push(prices, close)
if array.size(prices) > 500
    array.shift(prices)
```
Set `max_bars_back` only as high as needed; avoid deep `[N]` references with large N.

### 2.10 `Cannot use 'plot' in local scope`
See §7. `plot`/`hline`/`bgcolor`/`fill` must be global.

### 2.11 "Mismatched/qualifier" const errors on `hline`, `max_*_count`, `input` defaults
Those params demand compile-time constants — see §6 and §7.

---

## 3. Type System Traps

**Qualifier hierarchy (weak → strong):** `const` → `input` → `simple` → `series`. A param accepts its level **and weaker**; passing a **stronger** qualifier than required = compile error.

| Param expects | Accepts | Rejects |
|---|---|---|
| `const` | const | input, simple, series |
| `simple` | const, input, simple | series |
| `series` | everything | — |

```pine
// const-only example: hline price, max_*_count, plot title, indicator title
hline(50)                          // OK: literal const
levelInput = input.int(50)
hline(levelInput)                  // ERROR: input is not const for hline price
// -> hline price must be input int/float known on bar 0; use input.* directly:
hline(input.int(50, "Level"))      // RIGHT — input qualifies

// simple-demanding example
length = input.int(14)             // input -> accepted as simple
ta.atr(length)                     // OK
```

**`na` handling & `nz()`:**
```pine
prevClose = close[1]               // na on first bar
safe = nz(prevClose, close)        // default fallback
val = na(x) ? 0.0 : x              // explicit na guard (x must be int/float, NOT bool)
```

**int vs float:** mixing promotes to float. Force with `int()` (truncates toward 0), `math.round()`, `math.floor()`, `math.ceil()`. `float` literals: `1.0` not `1` when you need float typing in a tuple/UDT field.

**Type annotation:**
```pine
var float runningMax = na
int    count   = 0
bool   flag    = false
array<float> xs = array.new<float>()
```

---

## 4. Repainting & `request.security()`

### 4.1 The canonical non-repainting HTF pattern
Offset the expression by `[1]` **and** use `barmerge.lookahead_on`. Both are required together.
```pine
// RIGHT — confirmed HTF data, no repaint, no future leak
htfClose = request.security(syminfo.tickerid, "D", close[1],
     lookahead = barmerge.lookahead_on)

// Reusable wrapper
noRepaint(sym, tf, src) =>
    request.security(sym, tf, src[1], lookahead = barmerge.lookahead_on)
```

**Why:** `lookahead_on` alone pulls the *final* HTF value onto historical bars = future leak (looks amazing in backtest, fails live). `[1]` alone still repaints on the forming realtime HTF bar. Together: `[1]` selects the last *confirmed* HTF bar, `lookahead_on` lets you access it immediately and consistently on history and realtime.

```pine
// WRONG — repaints (uses live, unconfirmed HTF value)
htf = request.security(syminfo.tickerid, "D", close)

// WRONG — lookahead WITHOUT offset = future data on history (dangerous)
htf = request.security(syminfo.tickerid, "D", close,
     lookahead = barmerge.lookahead_on)
```

### 4.2 Gaps
Use `barmerge.gaps_off` (default) to forward-fill HTF values onto every chart bar; `gaps_on` returns `na` between HTF updates (then you must `nz`/handle na).

### 4.3 `barstate` gotchas
- `barstate.isconfirmed` is `true` only on a closed bar — gate signal logic on it to avoid intrabar repaint:
```pine
signal = cross and barstate.isconfirmed
```
- `barstate.isrealtime` / `barstate.ishistory` differ; logic that only runs in one will look different when realtime bars become history. Backtest and forward-test must be reconciled.
- For alerts that must fire only on closed bars, combine `alert()` with `barstate.isconfirmed`.

---

## 5. Strategy-Specific Behavior

### 5.1 Calculation timing
- **`process_orders_on_close=true`** — fill on the **close of the signal bar** instead of next bar's open. Removes the default 1-bar entry delay (and the look-ahead-ish boost it can give live; use carefully).
- **`calc_on_order_fills=true`** — extra intrabar recalculation right after each fill (for stop adjustment, fast pyramiding). In backtests this can fill multiple orders per bar (open/high/low/close points).
- **`calc_on_every_tick=true`** — recompute on each realtime tick. Historical results stay bar-close-based, so **live ≠ backtest** by design with this on.

```pine
strategy("S", overlay = true,
     process_orders_on_close = true,
     calc_on_order_fills     = false,
     calc_on_every_tick      = false,
     pyramiding              = 0,
     default_qty_type        = strategy.percent_of_equity,
     default_qty_value       = 10,
     margin_long             = 0,   // restore v5-like no-margin behavior
     margin_short            = 0)
```

### 5.2 Why backtest ≠ live
Default fills occur at **next bar open**; `calc_on_every_tick` and realtime ticks change intrabar order; HTF/repainting sources; broker slippage/commission not modeled unless set. Set `slippage` and `commission_value` to be realistic.

### 5.3 Pyramiding / sizing
`pyramiding=N` caps same-direction entries; beyond it, entries are ignored. `default_qty_type` ∈ `strategy.fixed` | `strategy.cash` | `strategy.percent_of_equity`, paired with `default_qty_value`. Per-order `qty=` overrides defaults.

### 5.4 Entry/exit surprises
- `strategy.exit()` needs a matching open entry `id`/`from_entry`; without it, nothing exits.
- In v6 `strategy.exit` evaluates both absolute and relative targets — don't double-specify unless intended.
- `strategy.close()` flattens by entry id; `strategy.close_all()` flattens everything.

### 5.5 Webhook alert plumbing
```pine
msg = '{"action":"buy","symbol":"' + syminfo.ticker +
      '","qty":' + str.tostring(qty) + '}'
strategy.entry("L", strategy.long, alert_message = msg)
```
Then in the **Create Alert** dialog, put **`{{strategy.order.alert_message}}`** as the message so the per-order string reaches the webhook. Other placeholders: `{{strategy.order.action}}`, `{{strategy.order.contracts}}`, `{{strategy.market_position}}`, `{{ticker}}`, `{{close}}`, `{{timenow}}`.

---

## 6. Drawing / Object Limits

- `line`, `box`, `label`: default **~50**, max **500**. `polyline`: max **100**.
- Set via `max_lines_count`, `max_boxes_count`, `max_labels_count`, `max_polylines_count` in the declaration. **Must be const integers**, range 1–500 (1–100 polylines); out-of-range or non-const → error.
- When the cap is hit, TradingView **silently deletes the oldest** objects. The cap is approximate (you may see a few extra).

```pine
//@version=6
indicator("X", overlay = true, max_labels_count = 500, max_boxes_count = 500)

// WRONG — leaks: a new label every bar, oldest silently dropped, slow
label.new(bar_index, high, "x")

// RIGHT — reuse a single persistent object, or delete before redraw
var label lbl = na
label.delete(lbl)
lbl := label.new(bar_index, high, "x")
```
Persist objects with `var`; track them in an `array<line>` and `.delete()` ones you no longer need to prevent leaks and stay under the cap.

```pine
// invalid value
indicator("X", max_lines_count = 600)   // ERROR: > 500
indicator("X", max_lines_count = 0)     // ERROR: < 1
n = input.int(100)
indicator("X", max_lines_count = n)     // ERROR: must be const, not input/series
```

---

## 7. Plotting Rules

- `plot()`, `hline()`, `bgcolor()`, `fill()`, `plotshape()`, `plotchar()`, `plotcandle()`, `plotbar()` must be in **global scope** (non-indented). No calling them inside `if`/`for`/functions.
- Plot **conditionally with values, not conditional calls**:
```pine
// WRONG — plot in local scope
if cond
    plot(close)                 // ERROR: Cannot use plot in local scope

// RIGHT — conditional value
plot(cond ? close : na, "Close")
bgcolor(cond ? color.new(color.green, 90) : na)
```
- `plot()` value must be `series int/float`; a `bool` needs converting (`b ? 1 : 0`).
- `plot` **title** requires a **const string** (compile-time).
- `hline()` price must be an **input/const** (known on bar 0), not series. For a dynamic level, use `plot()` instead of `hline()`.
- `plotshape`/`plotchar` are subject to a max number of shapes/labels rendered (TradingView throttles very dense shape plotting); prefer drawing objects with explicit caps for dense annotation.

---

## 8. Performance Limits

- **~40 unique `request.*()`** calls (64 Professional). Consolidate via tuples/UDTs (§2.8).
- **Per-bar compute budget (~500 ms/bar)** — exceeding it throws *"calculation takes too long"* / *"the script ... took too long to execute."* Avoid nested loops over large ranges per bar.
- **Loop limits** — extremely long/`while` loops can hit time/iteration ceilings; cap iterations, hoist invariant work out of loops, and pre-compute series once globally rather than recomputing in loops.
- **`max_bars_back`** — raise only when you reference deep history (`x[N]` with large N); too high inflates memory → *"Memory limits exceeded."*
- **Drawing objects** — keep well under 500 each; delete/reuse to avoid the per-bar churn of creating+dropping objects.
- **Hoist `ta.*` and `request.*`** to global scope (also avoids §2.7 and the lazy-eval trap §1.6) — both for correctness and speed.

```pine
// WRONG — O(n) request inside a loop, recomputed per bar
for i = 0 to 9
    v = request.security(syms[i], "D", close)   // many unique calls / slow

// RIGHT — request once into vars/arrays at global scope, loop over results
```

---

### One-glance "won't compile" checklist
1. `//@version=6` present. 2. No `bool ... = na`, no `na()/nz()` on bools. 3. No bare int/float in `if` — explicit comparison. 4. `transp` removed → `color.new()`. 5. Stateful `ta.*`/`request.*` hoisted to global scope. 6. `request.security` mutable args wrapped in a function; non-repaint = `src[1]` + `lookahead_on`. 7. `simple`/`const` params (lengths, `hline`, `max_*_count`, titles) not fed `series`. 8. `plot`/`hline`/`bgcolor` global, conditional via value (`cond ? x : na`). 9. ≤40 unique `request.*()` (tuple-consolidate). 10. Drawing caps ≤500, objects reused/deleted. 11. 4-space indents, no tabs, no comments inside wrapped lines, continuation indent not a multiple of 4. 12. `switch`/`if` returning unique types have a default/`else`.

Sources: [TradingView — To Pine Script v6 migration guide](https://www.tradingview.com/pine-script-docs/migration-guides/to-pine-version-6/), [TradingView — Repainting](https://www.tradingview.com/pine-script-docs/concepts/repainting/), [TradingView — Lines and boxes](https://www.tradingview.com/pine-script-docs/visuals/lines-and-boxes/), [TradingView — Plots](https://www.tradingview.com/pine-script-docs/visuals/plots/), [TradingView — Variable declarations](https://www.tradingview.com/pine-script-docs/language/variable-declarations/), [TradingCode — not find function](https://www.tradingcode.net/tradingview/not-find-function/), [TradingCode — mismatched input](https://www.tradingcode.net/tradingview/mismatched-input/), [TradingCode — too many securities](https://www.tradingcode.net/tradingview/request-too-many-securities/), [TradingCode — calc every tick](https://www.tradingcode.net/tradingview/calculate-every-tick/), [TradingCode — calc on order fills](https://www.tradingcode.net/tradingview/calculate-order-fills/), [TradingCode — max labels](https://www.tradingcode.net/tradingview/max-labels-setting/), [Pineify — plot in local scope](https://pineify.app/resources/blog/pine-script-cannot-use-plot-in-local-scope-complete-guide-to-fix-this-common-error), [TradersPost — v6 breaking changes the converter misses](https://blog.traderspost.io/article/pine-script-v6-breaking-changes), [CrossTrade — v6 strategy examples](https://crosstrade.io/blog/pine-script-v6-strategy-code-examples).
