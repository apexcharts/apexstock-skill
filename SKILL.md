---
name: apexstock
description: >
  AI skill for building ApexStock financial and stock charts. Use whenever the
  user asks to create, configure, render, update, stream, or troubleshoot a
  candlestick / OHLC / heikin-ashi price chart, add technical indicators (RSI,
  MACD, Bollinger Bands, moving averages, Ichimoku, and more), draw trading
  overlays (order / stop-loss / take-profit / alert price lines), aggregate
  timeframes, or feed a real-time tick stream with `apexstock`. Covers the
  OHLC data format (`{ x, y: [o,h,l,c], v? }`), the ApexCharts-global
  requirement, overlays-vs-oscillators, the `render()` / `update()` /
  `appendData()` / `destroy()` lifecycle, theming via `--apexstock-*` tokens,
  and framework integration (React / Vue / Angular). In React / Vue / Angular
  projects, prefer the framework wrapper packages (`react-apexstock`,
  `vue-apexstock`, `ngx-apexstock`) over the core API.
metadata:
  author: ApexCharts
  version: "1.0.1"
  library_version: "0.3.1"
  category: data-visualization
  tags: [stock, financial, candlestick, ohlc, technical-indicators, charts, apexstock]
  docs: https://apexcharts.com/docs/apexstock/
  npm: apexstock
  github: https://github.com/apexcharts/apexstock
---

# ApexStock AI Skill

ApexStock is a financial-charting **layer on top of ApexCharts**. It renders an
OHLC/candlestick main chart plus technical-indicator panes, drawing tools,
trading price lines, real-time streaming, and light/dark theming.

## Framework wrapper detection: check `package.json` first

- `react` → use **`react-apexstock`** instead of the core API.
- `vue` → use **`vue-apexstock`**.
- `@angular/core` → use **`ngx-apexstock`**.

The wrappers call `render()` on mount and `destroy()` on unmount, accept
`options` / `series` as reactive props, and expose the instance via
`getInstance()`. Use the core API directly only when no framework is detected,
or when the user explicitly asks for vanilla JS. **Even with a wrapper, the
ApexCharts-global rule below still applies**, the wrappers do not set it for
you.

---

## 1. Critical Rules

1. **ApexCharts must be a global before you construct.** ApexStock calls
   `new ApexCharts(...)` internally and does **not** import it. Set
   `window.ApexCharts = ApexCharts` first (or load ApexCharts via a UMD
   `<script>` before ApexStock). The constructor throws if `ApexCharts` is
   undefined.
2. **Always call `stock.render()`** after `new ApexStock(el, options)`. The
   constructor builds state; `render()` paints the chart and mounts the
   toolbar, drawing tools, and x-axis.
3. **OHLC data lives in `series[0].data`** as points shaped
   `{ x, y: [open, high, low, close], v? }`. The four prices are a **single
   `y` array**, not separate `o` / `h` / `l` / `c` keys. Points missing a valid
   `x` or a 4-number `y` are silently dropped.
4. **Sort data ascending by `x`.** `x` is an epoch-ms number, an ISO date
   string, or a `Date`. `v` (volume) is optional.
5. **Overlays vs oscillators.** Overlays (moving average, Bollinger Bands, EMA,
   Fibonacci, linear regression, Ichimoku) draw on the price chart and multiple
   can be active. Oscillators (RSI, MACD, volumes, stochastic, ADX, CCI, …)
   render in their own pane and **only one oscillator is active at a time**.
   Adding a second replaces the first.
6. **Indicator keys are lowercase full phrases**, e.g. `"moving average"`,
   `"bollinger bands"`, `"rsi"`, `"macd"`, `"ichimoku cloud indicator"`. Not
   abbreviations like `"ma"` or `"bb"`.
7. **For live data use `appendData()`, not `update()`.** `appendData` updates
   price, indicators, panes, and the x-axis incrementally (O(active indicators ×
   small tail)); `update()` does a full teardown/rebuild.
8. **`update(newOptions)` merges and preserves** zoom, active indicators,
   theme, and chart type. In framework wrappers, pass a **new object/array
   identity** to trigger an update (in-place mutation won't).
9. **`destroy()` has a teardown gap in 0.3.0**, it does not remove the
   indicator toolbar or the drawing-tools DOM (see Pitfalls §9). Give each chart
   its **own wrapper element** and remove that wrapper on teardown, especially
   in single-page apps.
10. **Theme** is `theme: { mode: 'light' | 'dark' }` at construction, or
    `stock.updateTheme('dark')` at runtime. Restyle the UI beyond light/dark by
    overriding `--apexstock-*` CSS custom properties (they are scoped and do not
    leak into your page).
11. **Set the license key once** at app startup with
    `ApexStock.setLicense('KEY')` to remove the watermark.
12. **Trading price lines persist** across `update()`, theme change,
    chart-type switch, and streaming appends. Add them with `addOrderLine` /
    `addStopLoss` / `addTakeProfit` / `addAlert`, or the generic
    `addPriceLine`.

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
indicators read. See [references/data-format.md](references/data-format.md) for
chart types (candlestick, ohlc, line, area, heikinashi), timestamp handling,
and normalization behavior.

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

window.ApexCharts = ApexCharts;                 // REQUIRED, ApexStock needs the global
ApexStock.setLicense('KEY');                    // optional, once at app startup

const stock = new ApexStock(document.getElementById('chart'), {
  chart: { height: 600 },
  series: [{ name: 'AAPL', data: candles }],    // candles: OHLCPoint[]
  theme: { mode: 'light' },
  plotOptions: { stockChart: { indicators: { 'moving average': { enabled: true } } } },
});
stock.render();                                 // REQUIRED

stock.updateIndicator('rsi');                   // add/toggle an indicator at runtime
stock.appendData({ x: t, y: [o, h, l, c], v }); // stream a new bar (see §7)
stock.update({ theme: { mode: 'dark' } });      // merge + re-render, preserving state
stock.destroy();                                // tear down, but see Pitfalls §9
```

---

## 5. Public API

| Method | Description |
|---|---|
| `render()` | Paint the chart and mount sub-components. Call once after construction. |
| `update(newOptions)` | Merge partial options and re-render, preserving zoom, active indicators, theme, and chart type. |
| `appendData(point \| point[], opts?)` | Incrementally append/replace bars for streaming. `opts`: `{ view?: 'follow'\|'preserve', maxPoints?, updateLast? }`. See §7. |
| `destroy()` | Tear down sub-components and listeners. **Does not remove the toolbar / drawing DOM in 0.3.0**, see §9. |
| `updateIndicator(key)` | Add or refresh an indicator overlay/oscillator (preserves zoom). |
| `removeIndicator(key)` | Remove an indicator overlay/oscillator. |
| `updateTheme('light' \| 'dark')` | Switch theme at runtime. |
| `getTheme()` | Current theme mode. |
| `addPriceLine(config)` | Generic trading price line; returns an id (or `null`). |
| `addOrderLine(config?)` / `addStopLoss(config?)` / `addTakeProfit(config?)` / `addAlert(config?)` | Typed price-line shortcuts. |
| `updatePriceLine(id, patch)` / `removePriceLine(id)` / `clearPriceLines()` | Manage price lines. |
| `getPriceLine(id)` / `getPriceLines()` | Read price-line configs. |
| `updateChartOptions(newOptions)` | Update chart options with theme handling. |
| `calculate*(series, …)` | Pure indicator math (`calculateRSI`, `calculateMACD`, `calculateBollingerBands`, `calculateEMA`, …) returning raw arrays, if you need the values yourself. |

**Static:**

| Static | Description |
|---|---|
| `ApexStock.setLicense(key)` | Register a license key globally (removes the watermark). |
| `ApexStock.aggregateOHLC(series, interval)` | Roll candles up to a coarser timeframe (pure helper). |
| `ApexStock.INTERVALS` | Accepted interval keys: `1m 5m 15m 30m 1h 2h 4h 12h 1d 1w 1M`. |

---

## 6. Technical Indicators

Configure at construction via `plotOptions.stockChart.indicators` (object map
recommended, string array also accepted), or add/remove at runtime with
`updateIndicator(key)` / `removeIndicator(key)`.

**Overlays** (on the price chart, multiple allowed): `"moving average"`,
`"bollinger bands"`, `"exponential moving average"`, `"fibonacci retracements"`,
`"linear regression"`, `"ichimoku cloud indicator"`.

**Oscillators** (own pane, one at a time): `"rsi"`, `"macd"`, `"volumes"`,
`"price volume trend"`, `"stochastic oscillator"`,
`"standard deviation indicator"`, `"average directional index"`,
`"chaikin oscillator"`, `"commodity channel index"`, `"trend strength index"`,
`"accelerator oscillator"`, `"bollinger bands %b"`, `"bollinger bands width"`.

Full list, config keys (`period`, `stdDev`), and the pure `calculate*` helpers
are in [references/indicators.md](references/indicators.md).

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

Roll fine bars up to a coarser timeframe with the pure helper:

```js
const hourly = ApexStock.aggregateOHLC(oneMinuteSeries, '1h');
```

Tick-to-bar recipes (append completed bars vs. `updateLast` forming candle) and
`view`/`maxPoints` semantics are in
[references/streaming-and-aggregation.md](references/streaming-and-aggregation.md).

---

## 8. Trading Overlays (price lines)

```js
const id = stock.addOrderLine({ price: 98.5, side: 'buy', label: 'Entry', draggable: true });
stock.addStopLoss({ price: 95 });
stock.addTakeProfit({ price: 104 });
stock.addAlert({ price: 100, onCross: (e) => notify(e.direction) });

stock.updatePriceLine(id, { price: 97 });   // reprice
stock.removePriceLine(id);
```

Config fields, drag/close callbacks (`onMove`, `onRemove`, `onCross`), and the
interactive drawing tools are in
[references/trading-overlays.md](references/trading-overlays.md).

---

## 9. Pitfalls & Anti-Patterns

1. **Forgetting `window.ApexCharts`.** Symptom: constructor throws "ApexCharts
   was not found." Fix: `window.ApexCharts = ApexCharts` (or a UMD script tag)
   before constructing, this is true in framework apps too.
2. **Flat `o`/`h`/`l`/`c` keys.** Wrong: `{ x, o, h, l, c }`. Right:
   `{ x, y: [o, h, l, c], v? }`. Bad points are dropped, so a "missing" candle
   usually means a malformed `y`.
3. **Using `update()` for every tick.** Slow (full rebuild). Use `appendData()`
   for streaming.
4. **Expecting multiple oscillators.** Only one oscillator pane is active at a
   time; adding another replaces it. Overlays can stack.
5. **Abbreviated indicator keys.** Use the full phrase (`"moving average"`, not
   `"ma"`). Unknown keys are no-ops.
6. **Mutating the same `options`/`series` object in a wrapper and expecting a
   redraw.** Wrappers diff by identity, pass a new object/array (memoize in
   React).
7. **Unsorted or duplicate-`x` data.** Sort ascending; a new bar with the
   last bar's `x` is treated as the forming bar (with `updateLast`).
8. **`theme: someUnsetVar` (i.e. `theme: undefined`).** ApexStock strips a
   nullish top-level `theme` so ApexCharts applies its default; but don't rely
   on passing an explicit `undefined`, omit the key instead.
9. **Assuming `destroy()` fully cleans the DOM (0.3.0).** `destroy()` removes
   the shared stylesheet (ref-counted) and tears down trading overlays,
   chart-switch, and oscillator settings, but it does **not** remove the
   indicator toolbar (inserted as a sibling before your container) or the
   drawing-tools toolbar/overlay. In an SPA that mounts/unmounts repeatedly,
   these accumulate. **Remediation:** render into a dedicated wrapper element
   you own, and on teardown call `stock.destroy()` then remove the wrapper:
   ```js
   const host = document.createElement('div');   // a wrapper you control
   const el = document.createElement('div');
   host.appendChild(el);
   mountPoint.appendChild(host);
   const stock = new ApexStock(el, options);
   stock.render();
   // teardown:
   stock.destroy();
   host.remove();                                 // takes the toolbar sibling with it
   ```

> **Historical note (already fixed, do not code around these):** In versions
> before 0.2.0 the injected stylesheet declared un-namespaced tokens like
> `--font-size-sm` on `:root`, clobbering host design tokens; as of 0.2.0 every
> token is prefixed `--apexstock-*` and scoped to `[class^="apexstock-"]`, so it
> no longer leaks. A 1-second `setInterval` overlay-sync poll was also removed in
> 0.2.0. Neither is a live gotcha in 0.3.0.

---

## 10. Theming

`theme.mode` (`'light'` / `'dark'`) sets the palette; `updateTheme(mode)`
switches at runtime. To restyle the toolbar/UI (accent, radii, colors) beyond
light/dark, override the `--apexstock-*` CSS custom properties, they are scoped
to `[class^="apexstock-"]` and never leak to or from your page's `:root`. See
[references/theming.md](references/theming.md).

---

## 11. Reference Routing Table

| File | Read it for |
|---|---|
| [data-format.md](references/data-format.md) | OHLC point shape, chart types (candlestick / ohlc / line / area / heikinashi), timestamps, normalization |
| [indicators.md](references/indicators.md) | Full overlay/oscillator list, keys, `period`/`stdDev` config, `calculate*` helpers |
| [streaming-and-aggregation.md](references/streaming-and-aggregation.md) | `appendData`, `view`/`maxPoints`/`updateLast`, forming candle, `aggregateOHLC`, `INTERVALS` |
| [trading-overlays.md](references/trading-overlays.md) | Price-line config, drag/close/cross callbacks, drawing tools |
| [framework-wrappers.md](references/framework-wrappers.md) | `react-apexstock`, `vue-apexstock`, `ngx-apexstock` props, refs, cleanup |
| [theming.md](references/theming.md) | Light/dark modes, `--apexstock-*` token reference, override recipe |
