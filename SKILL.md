---
name: apexstock
description: >
  AI skill for building ApexStock financial and stock charts. Use whenever the
  user asks to create, configure, render, update, stream, or troubleshoot a
  candlestick / OHLC / heikin-ashi / renko price chart, add technical
  indicators (RSI, MACD, Bollinger Bands, VWAP, ATR, Donchian / Keltner
  Channels, moving averages, Ichimoku, and more), draw trading overlays
  (order / stop-loss / take-profit / alert price lines), place data-space
  drawings (trend lines, rays, Fibonacci, measure) or annotations, compare
  symbols, export images or data, persist chart state, aggregate timeframes,
  or feed a real-time tick stream with `apexstock`. Covers the OHLC data
  format (`{ x, y: [o,h,l,c], v? }`), the data adapters, ApexCharts injection
  (per instance, app-wide, or the window global), overlays vs oscillators
  (multiple oscillator panes since 0.4.0), the `render()` / `update()` /
  `appendData()` / `destroy()` lifecycle, events, `getState` / `setState`,
  theming via `--apexstock-*` tokens, and framework integration
  (React / Vue / Angular). In React / Vue / Angular projects, prefer the
  framework wrapper packages (`react-apexstock`, `vue-apexstock`,
  `ngx-apexstock`) over the core API.
metadata:
  author: ApexCharts
  version: "1.1.0"
  library_version: "0.4.0"
  category: data-visualization
  tags: [stock, financial, candlestick, ohlc, technical-indicators, charts, apexstock]
  docs: https://apexcharts.com/docs/apexstock/
  npm: apexstock
  github: https://github.com/apexcharts/apexstock
---

# ApexStock AI Skill

ApexStock is a financial-charting **layer on top of ApexCharts** (6.x as of
ApexStock 0.4.0, `apexcharts` peer dependency `^6.7.0`). It renders an
OHLC/candlestick main chart plus technical-indicator panes (multiple
oscillators at once), interactive and programmatic drawing tools, data-space
annotations, trading price lines, multi-symbol comparison, real-time
streaming, state persistence, image/data export, and light/dark theming.

## Framework wrapper detection: check `package.json` first

- `react` → use **`react-apexstock`** instead of the core API.
- `vue` → use **`vue-apexstock`**.
- `@angular/core` → use **`ngx-apexstock`**.

The wrappers call `render()` on mount and `destroy()` on unmount, accept
`options` / `series` as reactive props, expose the instance via
`getInstance()`, and take an optional **`apexCharts` prop/input** to inject the
imported ApexCharts constructor (no `window` global needed). Use the core API
directly only when no framework is detected, or when the user explicitly asks
for vanilla JS.

---

## 1. Critical Rules

1. **ApexCharts is required but not bundled.** Provide the constructor one of
   three ways, resolved in this order: per instance
   (`new ApexStock(el, options, { ApexCharts })`), app-wide
   (`ApexStock.setApexCharts(ApexCharts)`), or the `window.ApexCharts` global
   (still fine for `<script>` tags). The constructor throws a clear error if
   none resolve. Injection is new in 0.4.0; before that, only the global
   worked.
2. **Always call `stock.render()`** after `new ApexStock(el, options)`. The
   constructor builds state; `render()` paints the chart and mounts the
   toolbar, drawing tools, and x-axis.
3. **OHLC data lives in `series[0].data`** as points shaped
   `{ x, y: [open, high, low, close], v? }`. The four prices are a **single
   `y` array**, not separate `o` / `h` / `l` / `c` keys. Points missing a valid
   `x` or a 4-number `y` are silently dropped. Convert other shapes with the
   static data adapters `ApexStock.normalize` / `fromArrays` / `fromCSV`.
4. **Sort data ascending by `x`.** `x` is an epoch-ms number, an ISO date
   string, or a `Date`. `v` (volume) is optional. Only `series[0]` is the
   instrument; use comparison mode for extra symbols, not extra series.
5. **Overlays and oscillators both stack (0.4.0).** Overlays (moving average,
   Bollinger Bands, VWAP, Donchian/Keltner Channels, Ichimoku, …) draw on the
   price chart. Oscillators (RSI, MACD, volumes, ATR, …) each render in their
   own pane, and **multiple oscillators can be active at once**: the panes
   share the indicator area evenly and resize on add/remove. (Before 0.4.0
   only one oscillator was active at a time.)
6. **Indicator keys are lowercase full phrases**, e.g. `"moving average"`,
   `"bollinger bands"`, `"rsi"`, `"vwap"`, `"ichimoku cloud indicator"`. Not
   abbreviations like `"ma"` or `"bb"`. Keys are case-insensitive; display
   labels are title-cased per word ("Bollinger Bands"), but keys and series
   names are unchanged.
7. **For live data use `appendData()`, not `update()`.** `appendData` updates
   price, indicators, panes, and the x-axis incrementally (O(active indicators ×
   small tail)); `update()` does a full teardown/rebuild.
8. **`update(newOptions)` merges and preserves** zoom, active indicators,
   theme, and chart type. In framework wrappers, pass a **new object/array
   identity** to trigger an update (in-place mutation won't).
9. **`updateIndicator(key)` toggles; `updateIndicator(key, params)` never
   toggles off.** The two-argument form (alias `setIndicatorParams(key,
   params)`) sets the params and ensures the indicator is active: an inactive
   indicator is added with the params, an active one is recomputed in place,
   preserving zoom.
10. **`destroy()` fully tears the chart down as of 0.4.0**: it destroys the
    main chart and every oscillator pane, removes all listeners/observers and
    the injected stylesheet, and is idempotent (safe before `render()` and
    under React StrictMode). Versions ≤ 0.3.x left toolbars behind; only there
    use the wrapper-element remediation (see Pitfalls).
11. **Theme** is `theme: { mode: 'light' | 'dark' }` at construction, or
    `stock.updateTheme('dark')` at runtime. Restyle the UI beyond light/dark by
    overriding `--apexstock-*` CSS custom properties (they are scoped and do not
    leak into your page).
12. **Set the license key once** at app startup with
    `ApexStock.setLicense('KEY')` to remove the watermark.
13. **Persistence boundaries.** Trading price lines persist across `update()`,
    theme change, chart-type switch, and streaming appends, and so do
    annotations and comparisons. `getState()` / `setState()` serializes theme,
    chart type, indicators + params, zoom, and **drawings** (schema v2);
    price lines and annotations are **not** serialized (they carry callbacks),
    persist them yourself via `getPriceLines()` / `getAnnotations()`.

---

## 2. OHLC Data Format (`series[0].data`)

```ts
type OHLCPoint = {
  x: number | string | Date;            // epoch ms | ISO string | Date; ascending
  y: [number, number, number, number];  // [open, high, low, close]
  v?: number;                            // optional volume
};
```

```js
series: [
  {
    name: 'AAPL',
    data: [
      { x: '2024-01-01', y: [100, 110, 95, 105], v: 1_000_000 },
      { x: '2024-01-02', y: [105, 115, 100, 112], v: 1_200_000 },
      // ...more bars, ascending by x
    ],
  },
];
```

Only the first series holds the OHLC data. The close (`y[3]`) is what most
indicators read. If your data arrives as row objects, parallel arrays, or CSV,
convert it with `ApexStock.normalize` / `fromArrays` / `fromCSV` (validated and
time-sorted output). See [references/data-format.md](references/data-format.md)
for chart types (candlestick, line, area, column, heikin-ashi, renko),
timestamp handling, normalization, the data adapters, and data export.

---

## 3. Top-Level Options (`StockChartOptions`)

```ts
{
  chart: { height?: number; id?: string; /* any ApexCharts chart config */ };
  series: Array<{ name?: string; data: OHLCPoint[] }>;   // series[0] = OHLC
  theme?: { mode?: 'light' | 'dark' };
  plotOptions?: {
    stockChart?: {
      // object map (recommended) OR array of keys
      indicators?: Record<string, { enabled?: boolean; period?: number; stdDev?: number }>
                 | string[];
    };
  };
}
```

The options object is a **superset of standard ApexCharts options**, the
`chart` block accepts the usual ApexCharts config. ApexStock reads the financial
data from `series[0].data` and its own settings from `plotOptions.stockChart`.

> ApexStock **mutates the options object** it receives (it sets `chart.id`,
> writes back normalized `series`, and strips a nullish `theme`). Pass it a
> plain object you don't mind being touched. The framework wrappers already
> hand it a shallow copy of `chart` and each series entry.

---

## 4. Lifecycle Pattern

```js
import ApexCharts from 'apexcharts';
import ApexStock from 'apexstock';

ApexStock.setLicense('KEY');                    // optional, once at app startup

const stock = new ApexStock(document.getElementById('chart'), {
  chart: { height: 600 },
  series: [{ name: 'AAPL', data: candles }],    // candles: OHLCPoint[]
  theme: { mode: 'light' },
  plotOptions: { stockChart: { indicators: { 'moving average': { enabled: true } } } },
}, { ApexCharts });                             // inject the constructor (or setApexCharts / window global)
stock.render();                                 // REQUIRED

stock.updateIndicator('rsi');                   // toggle an indicator at runtime
stock.updateIndicator('rsi', { period: 21 });   // set params + ensure active (never toggles off)
stock.appendData({ x: t, y: [o, h, l, c], v }); // stream a new bar (see §7)
stock.on('rangeChange', ({ min, max }) => {});  // subscribe to events (see §9)
stock.update({ theme: { mode: 'dark' } });      // merge + re-render, preserving state
stock.destroy();                                // full teardown (idempotent as of 0.4.0)
```

---

## 5. Public API

| Method | Description |
|---|---|
| `render()` | Paint the chart and mount sub-components. Call once after construction. |
| `update(newOptions)` | Merge partial options and re-render, preserving zoom, active indicators, theme, and chart type. |
| `appendData(point \| point[], opts?)` | Incrementally append/replace bars for streaming. `opts`: `{ view?: 'follow'\|'preserve', maxPoints?, updateLast? }`. See §7. |
| `destroy()` | Full teardown: charts, panes, listeners, observers, stylesheet, subscriptions. Idempotent. |
| `updateIndicator(key, params?)` | One arg: toggle the indicator. With `params`: set params and ensure active (see §6). |
| `setIndicatorParams(key, params)` | Explicit params form of the above (never toggles off). |
| `removeIndicator(key)` | Remove an indicator overlay/oscillator. |
| `listIndicators()` / `getIndicator(key)` | Introspect available indicators (built-in + custom) with `key`, `label`, `type`, `builtin`, `active`, `streamable`, `params`. |
| `on(name, handler)` / `off(name, handler?)` / `once(name, handler)` / `emit(name, payload)` | Event bus; `on` returns an unsubscribe function. See §9. |
| `getState()` / `setState(state)` | Serialize/restore theme, chart type, indicators + params, zoom, and drawings as versioned JSON. See §9. |
| `getVisibleRange()` / `setVisibleRange(min, max)` | Read or set the visible x-window across all panes (fires `rangeChange`). |
| `updateTheme('light' \| 'dark')` / `getTheme()` | Switch/read theme at runtime. |
| `addPriceLine(config)` | Generic trading price line; returns an id (or `null`). |
| `addOrderLine(config?)` / `addStopLoss(config?)` / `addTakeProfit(config?)` / `addAlert(config?)` | Typed price-line shortcuts. |
| `updatePriceLine(id, patch)` / `removePriceLine(id)` / `clearPriceLines()` / `getPriceLine(id)` / `getPriceLines()` | Manage price lines. |
| `addAnnotation(config)` / `updateAnnotation(id, patch)` / `removeAnnotation(id)` / `clearAnnotations()` / `getAnnotation(id)` / `getAnnotations()` | Data-space annotations: `yLine`, `yBand`, `xLine`, `xBand`, `point`, `text`. See §8. |
| `addDrawing(config)` / `updateDrawing(id, patch)` / `removeDrawing(id)` / `clearDrawings()` / `getDrawing(id)` / `getDrawings()` | Price/time-anchored drawings: trendline, ray, horizontalLine, verticalLine, rectangle, fibRetracement, fibExtension, measure. See §8. |
| `addComparison({ name, data, color? })` / `removeComparison(name)` / `clearComparisons()` / `getComparisons()` | Overlay other symbols on a secondary y-axis. See §8. |
| `setComparisonMode('percent' \| 'absolute')` / `getComparisonMode()` | Indexed %-change (default) vs raw prices. |
| `exportImage(opts?)` | `{ format: 'png'\|'svg', scale?, download?, filename? }` → `Promise<{ format, blob, url, fallback? }>`. PNG composites main chart + panes; falls back to SVG where rasterizing is blocked. |
| `exportData(opts?)` | `{ format: 'csv'\|'json', range: 'all'\|'visible', includeVolume?, raw?, pretty?, download?, filename? }` → string. Columns `time,open,high,low,close[,volume]`, ISO-8601 time. |
| `updateChartOptions(newOptions)` | Update chart options with theme handling. |
| `calculate*(series, …)` | Pure indicator math (`calculateRSI`, `calculateMACD`, `calculateBollingerBands`, `calculateEMA`, …) returning raw arrays, if you need the values yourself. |

**Static:**

| Static | Description |
|---|---|
| `ApexStock.setLicense(key)` | Register a license key globally (removes the watermark). |
| `ApexStock.setApexCharts(ctor)` | Register the ApexCharts constructor app-wide (bundler-friendly; see Rule 1). |
| `ApexStock.registerIndicator(name, def)` | Register a custom indicator globally (declarative `{ type, calc }` or advanced form, optional `stream` twin). See [indicators.md](references/indicators.md). |
| `ApexStock.registerDrawingTool(name, def)` | Register a custom data-space drawing type (`def.render(data, helpers)` → SVG element). See [trading-overlays.md](references/trading-overlays.md). |
| `ApexStock.normalize(rows, mapping?)` | Adapter: array of objects or `[x,o,h,l,c,v]` tuples → OHLC points. |
| `ApexStock.fromArrays(columns)` | Adapter: parallel column arrays (only `close` required). |
| `ApexStock.fromCSV(text, opts?)` | Adapter: CSV text (`{ delimiter?, header?, mapping? }`). |
| `ApexStock.aggregateOHLC(series, interval)` | Roll candles up to a coarser timeframe (pure helper). |
| `ApexStock.INTERVALS` | Accepted interval keys: `1m 5m 15m 30m 1h 2h 4h 12h 1d 1w 1M`. |
| `ApexStock.STATE_VERSION` / `ApexStock.migrateState(state)` | Current state schema version (`2`) and the up-front migration helper. |

---

## 6. Technical Indicators

Configure at construction via `plotOptions.stockChart.indicators` (object map
recommended, string array also accepted), or at runtime with
`updateIndicator(key)` (toggle) / `updateIndicator(key, params)` (set params +
ensure active) / `removeIndicator(key)`.

**Overlays** (on the price chart, multiple allowed): `"moving average"`,
`"exponential moving average"`, `"vwap"`, `"bollinger bands"`,
`"donchian channels"`, `"keltner channels"`, `"fibonacci retracements"`,
`"linear regression"`, `"ichimoku cloud indicator"`.

**Oscillators** (own pane; multiple at once, panes share the indicator area):
`"rsi"`, `"macd"`, `"volumes"`, `"price volume trend"`,
`"stochastic oscillator"`, `"standard deviation indicator"`,
`"average directional index"`, `"atr"`, `"chaikin oscillator"`,
`"commodity channel index"`, `"trend strength index"`,
`"accelerator oscillator"`, `"bollinger bands %b"`, `"bollinger bands width"`.

Full list, per-indicator params (`period`, `stdDev`, VWAP `source`, Keltner
`emaPeriod`/`atrPeriod`/`multiplier`, …), introspection
(`listIndicators()` / `getIndicator(key)`), custom indicators
(`registerIndicator`), and the pure `calculate*` helpers are in
[references/indicators.md](references/indicators.md).

---

## 7. Real-time Streaming & Aggregation

Use `appendData()` for live data, it is the fast path, not `update()`.

```js
// A completed bar arrived, append and ride the right edge.
stock.appendData({ x: t, y: [o, h, l, c], v }, { view: 'follow' });

// Fixed-width live window (drops oldest bars past 500).
stock.appendData(bar, { maxPoints: 500 });

// Forming candle: keep the in-progress bar live as ticks arrive.
stock.appendData(bar, { updateLast: true });   // replaces the last bar when x matches
```

All built-in indicators (including VWAP, ATR, Donchian, and Keltner) have exact
streaming twins, so their values stay correct under `appendData()` without a
full recompute.

Roll fine bars up to a coarser timeframe with the pure helper:

```js
const hourly = ApexStock.aggregateOHLC(oneMinuteSeries, '1h');
```

Tick-to-bar recipes (append completed bars vs. `updateLast` forming candle),
`view`/`maxPoints` semantics, and lazy-loading with
`getVisibleRange()` / `setVisibleRange()` + the `rangeChange` event are in
[references/streaming-and-aggregation.md](references/streaming-and-aggregation.md).

---

## 8. Trading Overlays, Annotations, Drawings & Comparison

Four distinct layers; pick the right one:

- **Price lines**: horizontal order/stop/target/alert levels with drag, close,
  and cross callbacks.
- **Annotations**: id-based marks at data coordinates, `yLine`, `yBand`,
  `xLine`, `xBand`, `point`, `text`. No callbacks, persist across
  update/theme/chart-type switches.
- **Drawings**: price/time-anchored analysis shapes (trend line, ray,
  horizontal/vertical line, rectangle, Fibonacci, measure), drawn with the
  mouse toolbar or via `addDrawing()`, reproject through zoom/pan/resize, and
  serialize through `getState()`.
- **Comparison mode**: other symbols as lines on a secondary y-axis
  (`percent` indexed performance by default, or `absolute` prices).

```js
const id = stock.addOrderLine({ price: 98.5, side: 'buy', label: 'Entry', draggable: true });
stock.addAlert({ price: 100, onCross: (e) => notify(e.direction) });

stock.addAnnotation({ type: 'yLine', y: 128.5, label: 'Resistance' });

stock.addDrawing({
  type: 'trendline',
  points: [{ x: '2024-02-01', y: 118 }, { x: '2024-04-15', y: 134 }],
  snap: 'close',                       // snap endpoints to bar closes
});

stock.addComparison({ name: 'MSFT', data: msftBars });   // secondary y-axis
stock.setComparisonMode('percent');                       // default
```

Config fields, drawing types and aliases, `snap`, `drawing*` events,
`registerDrawingTool`, annotation types, and comparison semantics are in
[references/trading-overlays.md](references/trading-overlays.md).

---

## 9. Events, State & Export

**Events.** Subscribe with `on(name, handler)` (returns an unsubscribe
function); `off`, `once`, and `emit` are also available. Safe to subscribe
before `render()`; all subscriptions drop on `destroy()`.

| Event | Payload |
|---|---|
| `crosshairMove` / `click` | `{ dataPointIndex, seriesIndex, x, ohlc, volume, nativeEvent }` (`dataPointIndex: -1` and null fields when not over a candle) |
| `rangeChange` | `{ min, max, source }`, `source` is `'zoom'` / `'pan'` / `'reset'` |
| `indicatorToggle` | `{ key, active }` |
| `drawingAdded` / `drawingUpdated` | `{ id, drawing }` |
| `drawingRemoved` | `{ id }` |
| `drawingsCleared` | `{}` |

**State.** `getState()` returns a schema-versioned, `JSON.stringify`-safe
snapshot: theme, chart type, active indicators with params, visible x-range,
and the full drawing set (schema v2, `ApexStock.STATE_VERSION === 2`).
`setState(state)` restores it after `render()`; older v1 states migrate
forward automatically (drawings default to empty), and
`ApexStock.migrateState(state)` migrates up-front. Price lines and annotations
are not captured, persist them via `getPriceLines()` / `getAnnotations()`.

**Export.**

```js
const { blob, url } = await stock.exportImage({ format: 'png', scale: 2 });
await stock.exportImage({ format: 'svg', download: true });

const csv = stock.exportData({ format: 'csv' });                       // all bars
stock.exportData({ format: 'json', range: 'visible', download: true });
```

PNG composites the price chart and every oscillator pane into one raster;
browsers that block rasterizing fall back to SVG with `fallback: true` on the
result. `exportData` CSV round-trips through `ApexStock.fromCSV`.

---

## 10. Pitfalls & Anti-Patterns

1. **Not providing ApexCharts.** Symptom: constructor throws "ApexCharts was
   not found." Fix: inject it (`new ApexStock(el, options, { ApexCharts })` or
   `ApexStock.setApexCharts(ApexCharts)`) or set `window.ApexCharts` before
   constructing.
2. **Flat `o`/`h`/`l`/`c` keys.** Wrong: `{ x, o, h, l, c }`. Right:
   `{ x, y: [o, h, l, c], v? }`. Bad points are dropped, so a "missing" candle
   usually means a malformed `y`. Use `ApexStock.normalize` and friends to
   convert other shapes.
3. **Using `update()` for every tick.** Slow (full rebuild). Use `appendData()`
   for streaming.
4. **Working around the old one-oscillator cap.** Since 0.4.0 oscillators are
   independent toggles: adding MACD does not evict RSI, do not remove one
   before adding another. (On ≤ 0.3.x, adding a second oscillator replaced
   the first.)
5. **Abbreviated indicator keys.** Use the full phrase (`"moving average"`, not
   `"ma"`). Unknown keys are no-ops.
6. **Adding a second symbol as another `series` entry.** Extra series entries
   are treated as indicators, not instruments. Use
   `addComparison({ name, data })` instead.
7. **Mutating the same `options`/`series` object in a wrapper and expecting a
   redraw.** Wrappers diff by identity, pass a new object/array (memoize in
   React).
8. **Unsorted or duplicate-`x` data.** Sort ascending; a new bar with the
   last bar's `x` is treated as the forming bar (with `updateLast`).
9. **`theme: someUnsetVar` (i.e. `theme: undefined`).** ApexStock strips a
   nullish top-level `theme` so ApexCharts applies its default; but don't rely
   on passing an explicit `undefined`, omit the key instead.
10. **Expecting `getState()` to capture everything.** It captures theme, chart
    type, indicators + params, zoom, and drawings. Price lines and annotations
    are not serialized; persist and re-apply them yourself.
11. **Custom drawings vanish after reload.** A serialized custom drawing type
    re-renders only if `ApexStock.registerDrawingTool(name, def)` has been
    called again in the new session before `setState()`.

> **Historical note (fixed, do not code around these in 0.4.0):** In ≤ 0.3.x,
> `destroy()` did not remove the indicator toolbar or the drawing-tools DOM,
> so SPAs had to wrap each chart in an owned element and remove the wrapper on
> teardown. As of 0.4.0 `destroy()` is a complete, idempotent teardown (charts,
> panes, listeners, observers, stylesheet, subscriptions), safe under React
> StrictMode. Likewise, the pre-0.2.0 un-namespaced CSS token leak and the
> one-oscillator cap (≤ 0.3.x) are gone.

---

## 11. Theming

`theme.mode` (`'light'` / `'dark'`) sets the palette; `updateTheme(mode)`
switches at runtime. To restyle the toolbar/UI (accent, radii, colors) beyond
light/dark, override the `--apexstock-*` CSS custom properties, they are scoped
to `[class^="apexstock-"]` and never leak to or from your page's `:root`. See
[references/theming.md](references/theming.md).

---

## 12. Reference Routing Table

| File | Read it for |
|---|---|
| [data-format.md](references/data-format.md) | OHLC point shape, chart types (candlestick / line / area / column / heikin-ashi / renko), timestamps, normalization, data adapters (`normalize` / `fromArrays` / `fromCSV`), `exportData` / `exportImage` |
| [indicators.md](references/indicators.md) | Full overlay/oscillator list (incl. VWAP, ATR, Donchian, Keltner), keys, params, multi-oscillator panes, `updateIndicator(key, params)`, introspection, `registerIndicator`, `calculate*` helpers |
| [streaming-and-aggregation.md](references/streaming-and-aggregation.md) | `appendData`, `view`/`maxPoints`/`updateLast`, forming candle, streaming twins, visible-range accessors, `aggregateOHLC`, `INTERVALS` |
| [trading-overlays.md](references/trading-overlays.md) | Price lines, data-space annotations, the drawing API (`addDrawing`, types, `snap`, `registerDrawingTool`, `drawing*` events), comparison mode |
| [framework-wrappers.md](references/framework-wrappers.md) | `react-apexstock`, `vue-apexstock`, `ngx-apexstock` props (incl. `apexCharts` injection), refs, cleanup |
| [theming.md](references/theming.md) | Light/dark modes, `--apexstock-*` token reference, override recipe |
