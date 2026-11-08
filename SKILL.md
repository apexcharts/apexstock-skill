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
  or feed a real-time tick stream with `apexstock`. Also covers the 0.5.0
  financial-analysis workspace: window statistics (`getRangeStats`), the
  drawdown pane, range measurements and the analysis panel, multi-instrument
  comparison with alignment / baseline policies / indexed / relative / ratio
  modes and a benchmark role, price-scale modes, theme presets, event markers,
  the on-chart data legend, toolbar customization, cross-chart sync, and
  unified `export()` including PDF. Covers the OHLC data
  format (`{ x, y: [o,h,l,c], v? }`), the data adapters, ApexCharts injection
  (per instance, app-wide, or the window global), overlays vs oscillators
  (multiple oscillator panes since 0.4.0), the `render()` / `update()` /
  `appendData()` / `destroy()` lifecycle, events, `getState` / `setState`,
  theming via `--apexstock-*` tokens, and framework integration
  (React / Vue / Angular). ApexStock 0.5.2 works with `apexcharts`
  `^7.1.0 || ^8.0.0` (0.5.0 and 0.5.1 accept 7.x only).
  In React / Vue / Angular projects, prefer the
  framework wrapper packages (`react-apexstock`, `vue-apexstock`,
  `ngx-apexstock`) over the core API.
metadata:
  author: ApexCharts
  version: "2.1.0"
  library_version: "0.5.2"
  category: data-visualization
  tags: [stock, financial, candlestick, ohlc, technical-indicators, charts, apexstock]
  docs: https://apexcharts.com/docs/apexstock/
  npm: apexstock
  github: https://github.com/apexcharts/apexstock
---

# ApexStock AI Skill

ApexStock is a financial-charting **layer on top of ApexCharts**. It renders an
OHLC/candlestick main chart plus technical-indicator panes (multiple
oscillators at once), interactive and programmatic drawing tools, data-space
annotations, trading price lines, event markers, an on-chart data legend,
multi-instrument comparison, a window-statistics engine with a drawdown pane
and range measurements, real-time
streaming, state persistence, image/data/PDF export, and light/dark theming
with named presets.

> **⚠️ Peer range: apexstock 0.5.2 accepts `apexcharts` `^7.1.0 || ^8.0.0`.**
> 0.5.0 and 0.5.1 accept `^7.1.0` only (0.4.0 needed `^6.7.0`, 0.3.x
> `^5.15.0`), so on ApexCharts 8 use apexstock 0.5.2 or later. The 7.1.0 floor
> (since 0.5.0) is where the crosshair fix for a zoomed chart with indicators
> lives, so a correct crosshair is not available below it. The three framework
> wrappers ship at 0.2.3 with the same range; a wrapper at 0.2.2 accepts
> ApexCharts 7 only and fails with ERESOLVE next to ApexCharts 8. ApexCharts 8.0
> moved unit, sunburst, violin, drilldown, waterfall, dumbbell and streamgraph
> out of its default bundle. ApexStock draws only line, area, rangeArea, bar and
> candlestick, which stay in it, so no extra import is needed.
> `apexcharts/features/measure` is still opt-in on the default bundle
> (ApexCharts 8's `apexcharts/full` includes it), and it matters only for the
> core measure-ruler interop in §9 (see
> [references/analysis.md](references/analysis.md)).

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
    leak into your page). To follow an Ignite UI theme on the page, import the
    free `apexstock/ignite-ui` add-on *(from 0.6.0)*; see §12.
12. **Set the license key once** at app startup with
    `ApexStock.setLicense('KEY')` to remove the watermark. ApexStock has no
    free tier: it is licensed from the Premium plan up, whatever the
    organisation's size. Without a valid key it renders with a trial
    watermark, which is for evaluation only.
13. **Persistence boundaries (rewritten in 0.5.0).** Trading price lines persist
    across `update()`, theme change, chart-type switch, and streaming appends,
    and so do annotations and comparisons. `getState()` / `setState()` now
    serializes theme **and its preset**, chart type, indicators + params, zoom,
    drawings, **annotations**, **price lines**, **event markers**, **price
    scale**, **pane height ratios**, and the **comparison setup** (still schema
    v2). Two things are deliberately left out and need your help on restore:
    a price line's interactive callbacks (`onCross` / `onMove` / `onRemove`),
    which you re-bind with `updatePriceLine(id, { onCross })`, and comparison
    **instrument data**, which is re-supplied via `addComparison` in response
    to the new `comparisonRestoreNeeded` event. Measurements need no key of
    their own: a measurement is a `measure` drawing, so it rides in `drawings`.
14. **Statistics are unrounded, in percent units, and never extrapolated
    (0.5.0).** `getRangeStats()` and friends return raw numbers for you to
    format; every percent-like value is in percent units (`20.42` means
    +20.42%); anything the data cannot support is `null` with the reason in
    `warnings`. An annualized return is omitted below `minAnnualizeDays`
    (default 30) and an annualized volatility is omitted for intraday bars
    unless you supply `periodsPerYear`. Counts are called **bars**, not trading
    sessions, because ApexStock owns no market calendar.
15. **The drawdown pane is an ordinary indicator (0.5.0).**
    `updateIndicator('drawdown')` toggles it like any oscillator, so it
    inherits pane stacking, shared zoom, theme rebuild, `getDataAt`, and
    state round-tripping. It measures on `analysis.drawdownBasis`, so the pane,
    `getRangeStats()`, and `getDrawdown()` cannot disagree.
16. **The default comparison baseline changed in 0.5.0** from each instrument's
    own first point to `"common"` (the first x where every instrument,
    the primary included, has data). With instruments on different calendars
    the old default made each line start at 0% on a different date, which
    overstated the newest listing. `setComparisonOptions({ baseline: 'own' })`
    restores it. A single instrument on a matching calendar renders identically
    either way.
17. **A comparison instrument may not take the primary series' name (0.5.0).**
    `addComparison({ name })` refuses `series[0]`'s name and the reserved
    `"__primary__"`; previously that filtered the price series off the chart.

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
  theme?: { mode?: 'light' | 'dark'; preset?: string; designSystem?: boolean; tokens?: boolean };  // preset: 0.5.0; designSystem, tokens: 0.6.0
  plotOptions?: {
    stockChart?: {
      // object map (recommended) OR array of keys
      indicators?: Record<string, { enabled?: boolean; period?: number; stdDev?: number }>
                 | string[];
    };
  };

  // --- all new in 0.5.0 ---

  // Primary price-axis scale mode.
  priceScale?: {
    mode?: 'linear' | 'logarithmic' | 'percent' | 'indexed';
    base?: number; logBase?: number; indexBase?: number;
  };

  // The primary toolbar: hide it, hide built-in sections, or inject controls.
  toolbar?: {
    show?: boolean;
    items?: { chartType?: boolean; indicators?: boolean; drawing?: boolean; download?: boolean };
  };

  // The on-chart data legend (OHLC + change + volume + overlay values at the crosshair).
  legend?: {
    show?: boolean;
    position?: 'top-left' | 'top-right' | 'bottom-left' | 'bottom-right';
    showChange?: boolean; showVolume?: boolean; showIndicators?: boolean;
  };

  // Statistics engine, measurement tool, analysis panel, and comparison defaults.
  analysis?: {
    source?: 'close' | 'open' | 'high' | 'low';    // what change/averages read
    drawdownBasis?: 'close' | 'intrabar';          // 'intrabar' = each low vs the running high
    periodsPerYear?: number;                        // annualization convention
    minAnnualizeDays?: number;                      // default 30; below it, no annualized return
    by?: 'auto' | 'index' | 'x';                    // how a bare number endpoint is read
    measure?: { snap?: boolean | 'open' | 'high' | 'low' | 'close'; label?: Function };
    panel?: boolean | { position?, metrics?, formatters?, title?, placeholder? };
    comparison?: {
      mode?: 'absolute' | 'percent' | 'indexed' | 'relative' | 'ratio';
      benchmark?: string;
      join?: 'union' | 'primary' | 'intersection';
      fill?: 'hold' | 'gap' | 'drop';
      baseline?: 'common' | 'own' | 'visible' | number;
      indexBase?: number; source?: 'close' | 'open' | 'high' | 'low';
      rebaseRatio?: boolean; resample?: boolean | null;
    };
  };

  // Per-pane layout, keyed by indicator key. Panes divide the indicator area
  // in proportion to heightRatio (1 for most, 1.4 for the drawdown pane).
  panes?: Record<string, { heightRatio?: number }>;
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

ApexStock.setLicense('KEY');                    // required in production; without a key the chart shows a trial watermark

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
stock.on('rangeChange', ({ min, max }) => {});  // subscribe to events (see §10)
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
| `on(name, handler)` / `off(name, handler?)` / `once(name, handler)` / `emit(name, payload)` | Event bus; `on` returns an unsubscribe function. See §10. |
| `getState()` / `setState(state)` | Serialize/restore theme + preset, chart type, indicators + params, zoom, drawings, annotations, price lines, event markers, price scale, pane ratios and the comparison setup as versioned JSON (not price-line callbacks or comparison instrument data). See §10. |
| `getVisibleRange()` / `setVisibleRange(min, max)` | Read or set the visible x-window across all panes (fires `rangeChange`). |
| `updateTheme('light' \| 'dark')` / `getTheme()` | Switch/read theme at runtime. |
| `addPriceLine(config)` | Generic trading price line; returns an id (or `null`). |
| `addOrderLine(config?)` / `addStopLoss(config?)` / `addTakeProfit(config?)` / `addAlert(config?)` | Typed price-line shortcuts. |
| `updatePriceLine(id, patch)` / `removePriceLine(id)` / `clearPriceLines()` / `getPriceLine(id)` / `getPriceLines()` | Manage price lines. |
| `addAnnotation(config)` / `updateAnnotation(id, patch)` / `removeAnnotation(id)` / `clearAnnotations()` / `getAnnotation(id)` / `getAnnotations()` | Data-space annotations: `yLine`, `yBand`, `xLine`, `xBand`, `point`, `text`. See §8. |
| `addDrawing(config)` / `updateDrawing(id, patch)` / `removeDrawing(id)` / `clearDrawings()` / `getDrawing(id)` / `getDrawings()` | Price/time-anchored drawings: trendline, ray, horizontalLine, verticalLine, rectangle, fibRetracement, fibExtension, measure. See §8. |
| `addComparison({ name, data, color? })` / `removeComparison(name)` / `clearComparisons()` / `getComparisons()` | Overlay other instruments on a secondary y-axis. See §8. |
| `setComparisonMode(mode)` / `getComparisonMode()` | *(extended 0.5.0)* `'absolute'`, `'percent'` (default), `'indexed'`, `'relative'`, `'ratio'`. |
| `setComparisonBenchmark(name)` / `getComparisonBenchmark()` | *(0.5.0)* Point the benchmark role at any added instrument or `'__primary__'` (the default). |
| `setComparisonOptions(patch)` / `getComparisonOptions()` | *(0.5.0)* `{ join, fill, baseline, indexBase, source, rebaseRatio, resample }`. |
| `getComparisonStats(opts?)` | *(0.5.0)* The leaderboard, ready to render: one row per instrument (primary included) with change, excess return vs the benchmark, high, low, volatility, worst drawdown, rank, and grid coverage. |
| `setPriceScale(mode, opts?)` / `getPriceScale()` | *(0.5.0)* `'linear'` / `'logarithmic'` (opt. `logBase`) / `'percent'` (opt. `base`) / `'indexed'` (opt. `indexBase`, default 100). |
| `setPaneHeightRatio(key, ratio)` / `getPaneHeightRatios()` | *(0.5.0)* Per-pane share of the indicator area; `null` restores that pane's default. |
| `getDataAt(index?)` | *(0.5.0)* Read-only snapshot at a data-point index: OHLC, volume, change vs the previous close, and every active indicator's value across overlays and panes (`{ name, value, color, pane, key? }`). Defaults to the latest bar. |
| `getRangeStats(from, to, opts?)` | *(0.5.0)* Every statistic for a region: change, bars, calendar span, annualized return, true high/low, average close and volume, volatility, and the deepest drawdown within it. Endpoints in either order as a bar index, epoch-ms `x`, `Date`, or date string. |
| `getDrawdown(opts?)` | *(0.5.0)* Per-bar drawdown as a plottable series, plus every episode, the deepest, and the current one. `basis: 'intrabar'` measures each bar's low against the running high. |
| `measureRange(from, to, opts?)` / `getMeasurements()` / `getMeasurement(id)` / `clearMeasurement(id?)` | *(0.5.0)* Persistent range measurements. A measurement **is** a `measure` drawing, so it selects, drags, reshapes, and round-trips through `getState()`. |
| `showAnalysisPanel(opts?)` / `hideAnalysisPanel()` / `isAnalysisPanelVisible()` | *(0.5.0)* The on-chart region-statistics readout. Automatic by default. |
| `addEventMarker(config)` / `updateEventMarker(id, patch)` / `removeEventMarker(id)` / `clearEventMarkers()` / `getEventMarker(id)` / `getEventMarkers()` | *(0.5.0)* Time-anchored flags (`earnings` / `dividend` / `split` / `news` / `custom`) with a hover card. |
| `showLegend(opts?)` / `hideLegend()` / `toggleLegend()` / `isLegendVisible()` | *(0.5.0)* The on-chart OHLC + change + volume + overlay readout at the crosshair. |
| `addToolbarItem(def)` / `removeToolbarItem(id)` / `getToolbarItems()` | *(0.5.0)* Inject custom toolbar controls (`{ id, title, icon \| html \| element, onClick, position, order }`). |
| `setThemePreset(name)` / `getThemePreset()` | *(0.5.0)* Apply a named preset on top of the base light/dark mode. |
| `export(opts?)` | *(0.5.0)* Unified export over `png` / `svg` / `pdf` / `csv` / `json` → `Promise<{ format, blob, url, text? }>`. Adds **PDF** and the `include` selector. |
| `exportImage(opts?)` | `{ format: 'png'\|'svg', scale?, download?, filename? }` → `Promise<{ format, blob, url, fallback? }>`. PNG composites main chart + panes; falls back to SVG where rasterizing is blocked. |
| `exportData(opts?)` | `{ format: 'csv'\|'json', range: 'all'\|'visible', includeVolume?, include?, raw?, pretty?, download?, filename? }` → string. Columns `time,open,high,low,close[,volume]`, ISO-8601 time. `include` *(0.5.0)* appends `"indicators"` and/or `"analysis"` columns. |
| `updateChartOptions(newOptions)` | Update chart options with theme handling. |
| `calculate*(series, …)` | Pure indicator math (`calculateRSI`, `calculateMACD`, `calculateBollingerBands`, `calculateEMA`, …) returning raw arrays, if you need the values yourself. |

**Static:**

| Static | Description |
|---|---|
| `ApexStock.setLicense(key)` | Register a license key globally (removes the watermark). |
| `ApexStock.setApexCharts(ctor)` | Register the ApexCharts constructor app-wide (bundler-friendly; see Rule 1). |
| `ApexStock.registerIndicator(name, def)` | Register a custom indicator globally (declarative `{ type, calc }` or advanced form, optional `stream` twin). See [indicators.md](references/indicators.md). |
| `ApexStock.registerDrawingTool(name, def)` | Register a custom data-space drawing type (`def.render(data, helpers)` → SVG element). See [trading-overlays.md](references/trading-overlays.md). |
| `ApexStock.registerTheme(name, def)` / `ApexStock.getThemePresets()` | *(0.5.0)* Register a named preset from any subset of `{ mode, up, down, grid, axis, background, accent }` (the rest backfilled from the mode), and list what resolves. |
| `ApexStock.sync(instances, opts?)` | *(0.5.0)* Link independent instances so pan/zoom mirrors and a crosshair on one draws a guide on the others. `opts`: `{ zoom?, crosshair?, crosshairColor? }`. Returns a handle with `disconnect()`. |
| `ApexStock.stats` | *(0.5.0)* The whole statistics engine with no chart or DOM required (`rangeStats`, `drawdown`, `returns`, `volatility`, `annualize`, `inferPeriodsPerYear`, `align`, `baseline`, `rebase`, `relative`), for server-side reports, tests, and workers. |
| `ApexStock.normalize(rows, mapping?)` | Adapter: array of objects or `[x,o,h,l,c,v]` tuples → OHLC points. |
| `ApexStock.fromArrays(columns)` | Adapter: parallel column arrays (only `close` required). |
| `ApexStock.fromCSV(text, opts?)` | Adapter: CSV text (`{ delimiter?, header?, mapping? }`). |
| `ApexStock.aggregateOHLC(series, interval)` | Roll candles up to a coarser timeframe (pure helper). |
| `ApexStock.INTERVALS` | Accepted interval keys: `1m 5m 15m 30m 1h 2h 4h 12h 1d 1w 1M`. |
| `ApexStock.STATE_VERSION` / `ApexStock.migrateState(state)` | Current state schema version (`2`) and the up-front migration helper. |

`stock.Utils.compactNumber(value)` *(0.5.0)* is an **instance** property, not a static: `1.20B` / `3.40M` / `5.60K`, the formatter the legend's volume row and the analysis panel share.

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

**Also chart furniture (all new in 0.5.0):** **event markers** (time-anchored
`earnings` / `dividend` / `split` / `news` / `custom` flags with a hover card),
the **on-chart data legend** (OHLC, change, volume and overlay values at the
crosshair), and **toolbar customization** (hide the toolbar or its sections,
inject your own buttons). All three are in
[references/trading-overlays.md](references/trading-overlays.md).

```js
stock.addEventMarker({ x: '2024-02-01', type: 'earnings', label: 'Q4 beat' });
stock.showLegend({ position: 'top-left' });
stock.addToolbarItem({ id: 'snap', title: 'Snapshot', icon: '📷', onClick: (c) => c.export({ format: 'png' }) });
```

---

## 9. Analysis: Statistics, Drawdown, Measurements & Comparison (0.5.0)

The financial-analysis workspace. Four surfaces over one statistics engine, so
the pane, the panel, the export column, and the returned object can never
disagree.

```js
// 1. Window statistics over any region.
const stats = stock.getRangeStats('2024-01-02', '2024-03-15');
stats.change.percent;      // +20.42  (percent units, unrounded)
stats.drawdown.max;        // -6.4
stats.volatility;          // annualized, or null with the reason in stats.warnings

// 2. Drawdown, as data or as a pane.
const dd = stock.getDrawdown({ basis: 'intrabar' });
stock.updateIndicator('drawdown');                  // toggles the pane like any oscillator
stock.setPaneHeightRatio('drawdown', 2);            // panes split the area by ratio

// 3. A persistent, draggable measurement (and the analysis panel that reads it out).
const { id, stats: m } = stock.measureRange('2024-01-02', '2024-03-15');

// 4. Comparison that answers "who's up more" fairly.
stock.addComparison({ name: 'MSFT', data: msftBars });
stock.setComparisonBenchmark('MSFT');
stock.setComparisonMode('relative');                // percentage points vs the benchmark
const board = stock.getComparisonStats();           // the leaderboard, ready to render

// 5. Every active indicator's value at one bar.
const readout = stock.getDataAt(dataPointIndex);

// 6. The whole engine, headless. No chart, no DOM.
ApexStock.stats.rangeStats(series, '2024-01-02', '2024-03-15');
ApexStock.stats.align({ AAPL, SPY }, { join: 'intersection' });
```

Four contracts hold throughout: values are **unrounded** (you format them),
every percent-like value is in **percent units**, anything the data cannot
support is **`null` with the reason in `warnings`**, and counts are **bars**,
not trading sessions.

Field-by-field statistics, the drawdown pane and per-pane heights, the
measurement tool and analysis panel, ApexCharts measure-ruler interop, the full
comparison surface (alignment `join` / `fill`, the four baseline policies, the
five modes, the benchmark role, `getComparisonStats()` rows, the
`comparisonChange` event), `getDataAt`, price-scale modes, and the headless
`ApexStock.stats` namespace are in
[references/analysis.md](references/analysis.md).

---

## 10. Events, State & Export

**Events.** Subscribe with `on(name, handler)` (returns an unsubscribe
function); `off`, `once`, and `emit` are also available. Safe to subscribe
before `render()`; all subscriptions drop on `destroy()`.

| Event | Payload |
|---|---|
| `crosshairMove` / `click` | `{ dataPointIndex, seriesIndex, x, ohlc, volume, nativeEvent }` (`dataPointIndex: -1` and null fields when not over a candle) |
| `rangeChange` | `{ min, max, source }`, `source` is `'zoom'` / `'pan'` / `'reset'`. **Once per gesture**: a wheel or pinch zoom emits it when the gesture settles. The right rhythm for fetching data or writing the range to a URL. |
| `rangeChanging` *(0.5.0)* | Same payload with `source: 'live'`, fired on **every animation frame** of an in-progress zoom or pan. For overlays that must stay glued to the axis while the gesture runs. |
| `indicatorToggle` | `{ key, active }` |
| `drawingAdded` / `drawingUpdated` | `{ id, drawing }` |
| `drawingRemoved` | `{ id }` |
| `drawingsCleared` | `{}` |
| `eventMarkerAdded` / `eventMarkerUpdated` *(0.5.0)* | `{ id, marker }` |
| `eventMarkerRemoved` / `eventMarkersCleared` *(0.5.0)* | `{ id }` / `{}` |
| `eventMarkerHover` / `eventMarkerClick` *(0.5.0)* | `{ id, marker, nativeEvent }` |
| `priceScaleChange` *(0.5.0)* | `{ mode, base, logBase, indexBase }` |
| `rangeMeasured` *(0.5.0)* | `{ id, from, to, selection, stats, source }`, `source` is `'drag'` / `'api'` / `'coreRuler'`. Fires once per **settled** change: one drag is one event, and a plain zoom or pan produces none. |
| `measurementRemoved` *(0.5.0)* | `{ id }` |
| `comparisonChange` *(0.5.0)* | `{ reason, mode, benchmark, baseline, instruments, stats, warnings }`, carrying the recomputed leaderboard. Skipped entirely when nothing is subscribed. `baseline` reports the policy actually **applied**, which can fall back to `'own'`. |
| `comparisonRestoreNeeded` *(0.5.0)* | `{ names }` after `setState` restored a comparison whose instrument data is not loaded. Re-supply it with `addComparison`. |

**State.** `getState()` returns a schema-versioned, `JSON.stringify`-safe
snapshot (still schema v2, `ApexStock.STATE_VERSION === 2`). As of 0.5.0 it
captures theme **and its named preset**, chart type, active indicators with
params, visible x-range, drawings, **annotations**, **price lines**, **event
markers**, **price scale**, **pane height ratios**, and the **comparison**
setup. `setState(state)` restores it after `render()`; older states migrate
forward automatically, and `ApexStock.migrateState(state)` migrates up-front.

Two things are deliberately **not** captured:

- **Price-line callbacks** (`onCross` / `onMove` / `onRemove`) are not
  serializable. Re-bind them after `setState` with
  `updatePriceLine(id, { onCross })`.
- **Comparison instrument data.** Your app fetches it, it runs to thousands of
  bars per instrument, and it would be stale the moment it hit storage. So
  `setState` restores the setup, keeps any instrument still loaded, and emits
  `comparisonRestoreNeeded` with `{ names }` for the rest. The remembered color
  is reapplied when you hand the data back. A benchmark whose instrument has
  not returned yet is remembered by name, with the primary filling the role
  until it does.

Measurements need no state key: a measurement **is** a `measure` drawing, so it
round-trips inside `drawings`.

**Export.** `export({ format })` *(0.5.0)* is the unified entry point over
`png` / `svg` / `pdf` / `csv` / `json`, returning
`Promise<{ format, blob, url, text? }>`. `exportImage` and `exportData` remain
available and unchanged.

```js
const { blob, url } = await stock.export({ format: 'png', scale: 2 });
await stock.export({ format: 'pdf', include: ['analysis'], range: 'visible' });

const csv = stock.exportData({ format: 'csv' });                       // all bars
stock.exportData({ format: 'csv', include: ['indicators', 'analysis'] });
stock.exportData({ format: 'json', range: 'visible', download: true });
```

`include` *(0.5.0)* carries the analysis into an export and means something
appropriate to each medium:

| `include` | `csv` / `json` | `pdf` |
|---|---|---|
| `'indicators'` | One column per active indicator series (main-chart overlays **and** oscillator panes), `null` through each warm-up. | n/a |
| `'analysis'` | Adds `return` and `drawdown` per bar. | A text summary below the chart image: the window's dates and bar count, change, high, low, average, volume, annualized return, volatility, max drawdown, and the comparison leaderboard when one is active. Override with `summary: string[]`. |

The **OHLC spine is always present**, so a CSV keeps round-tripping through
`ApexStock.fromCSV` whatever you add; an extra column whose name collides with
a spine column is suffixed rather than overwriting it. Range statistics are a
summary, not a per-bar value, so they are not columns: read them from
`getRangeStats()`. The PDF summary follows `range`, defaulting to the visible
window, so the exported numbers describe what you were looking at. PDF is built
with no external library, and its summary is set in Helvetica (a PDF base-14
font), so there is nothing to embed.

PNG composites the price chart and every oscillator pane into one raster;
browsers that block rasterizing fall back to SVG with `fallback: true` on the
result.

Details, plus `ApexStock.sync()` for linking independent charts, are in
[references/state-and-export.md](references/state-and-export.md).

---

## 11. Pitfalls & Anti-Patterns

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
10. **Expecting `getState()` to capture everything.** As of 0.5.0 it captures
    a great deal more (theme + preset, chart type, indicators + params, zoom,
    drawings, annotations, price lines, event markers, price scale, pane
    ratios, comparison setup), but **not** price-line callbacks and **not**
    comparison instrument data. Re-bind the callbacks with
    `updatePriceLine(id, { onCross })` and answer `comparisonRestoreNeeded`
    with `addComparison`. Docs written against 0.4.0 said annotations and
    price lines were not captured at all; that is no longer true.
11. **Custom drawings vanish after reload.** A serialized custom drawing type
    re-renders only if `ApexStock.registerDrawingTool(name, def)` has been
    called again in the new session before `setState()`.
12. **Installing ApexStock 0.5.x against ApexCharts 6.** *(0.5.0)* The peer is
    `^7.1.0 || ^8.0.0` as of 0.5.2 (`^7.1.0` on 0.5.0 and 0.5.1);
    `npm install` fails with ERESOLVE, or, if forced, the crosshair drifts off
    the cursor on a zoomed chart with indicators. Upgrade ApexCharts.
13. **Subscribing to `rangeChange` for per-frame work.** *(0.5.0)* It is
    deliberately once per gesture. An overlay that must track a live zoom wants
    the new `rangeChanging` (`source: 'live'`); `rangeChange` is for the work
    that should happen once, like fetching data for the new window.
14. **Assuming the old comparison baseline.** *(0.5.0)* The default is now
    `'common'`, not each instrument's own first point. If your app's numbers
    shifted after upgrading, that is why; `setComparisonOptions({ baseline:
    'own' })` restores the previous behavior.
15. **Rounding twice.** *(0.5.0)* Statistics come back unrounded and in percent
    units. `stats.change.percent` is `20.42`, not `0.2042`, so do not multiply
    by 100 as well.
16. **Treating a `null` statistic as an error.** *(0.5.0)* It is the documented
    contract for "the data cannot support this": an annualized return under
    `minAnnualizeDays`, an annualized volatility on intraday bars with no
    `periodsPerYear`. The reason is in `warnings`.
17. **Naming a comparison instrument after the primary series.** *(0.5.0)*
    `addComparison` refuses it (and the reserved `'__primary__'`). Previously
    this silently filtered the price series off the chart.
18. **Expecting `getPanels`-style per-bar stats from an export.** Range
    statistics are a summary, not a column. `include: ['analysis']` gives you
    per-bar `return` and `drawdown`; the window summary comes from
    `getRangeStats()`, or from the PDF summary band.
19. **Installing ApexCharts 8 next to apexstock 0.5.0 / 0.5.1 or a 0.2.2
    wrapper.** *(0.5.2)* Those peer `apexcharts` `^7.1.0` only, so
    `npm install` fails with ERESOLVE. Upgrade to apexstock 0.5.2 and the
    wrapper to 0.2.3 rather than forcing the install or downgrading ApexCharts.
20. **Expecting an Ignite UI theme without the add-on.** *(0.6.0)* An Ignite UI
    stylesheet on the page changes nothing until you `import
    'apexstock/ignite-ui'`: the `--ig-*` tokens are ignored silently. And a
    `theme: { mode: 'light' }` (or a preset) pins the look, so a dark Ignite UI
    theme cannot turn the chart dark: leave `theme.mode` unset to follow it.

> **Historical note (fixed, do not code around these in 0.4.0):** In ≤ 0.3.x,
> `destroy()` did not remove the indicator toolbar or the drawing-tools DOM,
> so SPAs had to wrap each chart in an owned element and remove the wrapper on
> teardown. As of 0.4.0 `destroy()` is a complete, idempotent teardown (charts,
> panes, listeners, observers, stylesheet, subscriptions), safe under React
> StrictMode. Likewise, the pre-0.2.0 un-namespaced CSS token leak and the
> one-oscillator cap (≤ 0.3.x) are gone.

---

## 12. Theming

Three layers, in increasing order of specificity:

1. **Mode.** `theme: { mode: 'light' | 'dark' }` at construction, or
   `updateTheme(mode)` at runtime.
2. **Preset** *(0.5.0)*. `theme: { preset: 'mint' }`, or `setThemePreset(name)`
   / `getThemePreset()`. A curated light-first pack (`paper`, `arctic`, `mint`,
   `linen`, `rose`, `graphite`) layered on the base mode. Each preset retints
   **both** surfaces at once: the chart (candles, grid, axis labels, plot
   background) and the chrome (toolbar, dropdowns, legend, via the `--apx-*`
   tokens it sets on the container). Price lines follow the accent; the
   indicator palette stays the base mode's. A preset carries its own base mode,
   survives theme and chart-type switches and data updates, is captured by
   `getState()` as `theme.preset`, and is **cleared by `updateTheme()`**.
   Register your own with `ApexStock.registerTheme(name, def)` from any subset
   of `{ mode, up, down, grid, axis, background, accent }`.
3. **CSS tokens.** Override the `--apexstock-*` custom properties. They are
   scoped to `[class^="apexstock-"]` and never leak to or from your page's
   `:root`.

**Ignite UI theming** *(from apexstock 0.6.0; free, no licence key)*. An
opt-in add-on: load it once and stock charts follow the Ignite UI theme on the
page (toolbar, menus and popups, legend, analysis panel, rise and fall colours
from the theme's success and error colours, the font, and light or dark).

```js
import ApexCharts from 'apexcharts';
import ApexStock from 'apexstock';
import 'apexstock/ignite-ui';   // also loads apexcharts/features/ignite-ui
```

```html
<!-- script tags, in this order: ApexCharts' add-on themes the charts, ApexStock's the rest -->
<script src="apexcharts/dist/apexcharts.min.js"></script>
<script src="apexcharts/dist/features/ignite-ui.js"></script>
<script src="apexstock/dist/apexstock.min.js"></script>
<script src="apexstock/dist/apexstock.ignite-ui.min.js"></script>
```

- Ranking: everything you set wins (options, `--apexstock-*` and `--apx-*`
  tokens, a preset, candle colours passed to `update()`), then Ignite UI, then
  the defaults. A preset, colour-blind mode or monochrome keeps ApexStock's own
  colours.
- With no `theme.mode`, a dark Ignite UI theme makes the chart dark.
- `theme: { designSystem: false }` turns it off for one chart; `theme: {
  tokens: false }` also turns off the charts' `--apx-*` token reads.
- Without the add-on the `--ig-*` tokens are ignored; `--apx-*` work either way.
  Import it before the first render, or call `stock.update({})` after; the same
  call picks up a swapped Ignite UI stylesheet (ApexStock has no
  `refreshTokens()`).
- The bundler entry needs an ApexCharts release that ships
  `apexcharts/features/ignite-ui` (8.8.0 and later).

See [references/theming.md](references/theming.md).

---

## 13. Reference Routing Table

| File | Read it for |
|---|---|
| [data-format.md](references/data-format.md) | OHLC point shape, chart types (candlestick / line / area / column / heikin-ashi / renko), timestamps, normalization, data adapters (`normalize` / `fromArrays` / `fromCSV`), `exportData` / `exportImage` |
| [indicators.md](references/indicators.md) | Full overlay/oscillator list (incl. VWAP, ATR, Donchian, Keltner), keys, params, multi-oscillator panes, `updateIndicator(key, params)`, introspection, `registerIndicator`, `calculate*` helpers |
| [streaming-and-aggregation.md](references/streaming-and-aggregation.md) | `appendData`, `view`/`maxPoints`/`updateLast`, forming candle, streaming twins, visible-range accessors, `aggregateOHLC`, `INTERVALS` |
| [trading-overlays.md](references/trading-overlays.md) | Price lines, data-space annotations, the drawing API (`addDrawing`, types, `snap`, reshapeable two-anchor drawings, `registerDrawingTool`, `drawing*` events), and the 0.5.0 chart furniture: event markers, the on-chart data legend, toolbar customization |
| [framework-wrappers.md](references/framework-wrappers.md) | `react-apexstock`, `vue-apexstock`, `ngx-apexstock` props (incl. `apexCharts` injection), refs, cleanup |
| [theming.md](references/theming.md) | Light/dark modes, the 0.5.0 named preset pack + `registerTheme`, `--apexstock-*` token reference, override recipe, the 0.6.0 Ignite UI add-on |
| [analysis.md](references/analysis.md) | *(0.5.0)* `getRangeStats` field by field, `getDrawdown`, the drawdown pane and per-pane heights, `measureRange` + the analysis panel + measure-ruler interop, comparison v2 (alignment, baselines, the five modes, the benchmark role, `getComparisonStats`), `getDataAt`, price-scale modes, and the headless `ApexStock.stats` namespace |
| [state-and-export.md](references/state-and-export.md) | *(0.5.0)* The v2 state shape field by field, what is and is not captured and how to restore it, unified `export()` incl. PDF and `include`, and `ApexStock.sync()` |
