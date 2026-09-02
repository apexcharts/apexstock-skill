# Technical Indicators

ApexStock ships 23 built-in indicators, split into **overlays** (drawn on the
price chart) and **oscillators** (rendered in their own pane below the chart).
As of 0.4.0 **multiple oscillators can be active at once**, and you can add
your own indicators with `ApexStock.registerIndicator`.

## Two ways to configure

**1. At construction** via `plotOptions.stockChart.indicators`, an object map
(recommended) or an array of keys:

```js
// Object map, set enabled + per-indicator params.
plotOptions: {
  stockChart: {
    indicators: {
      'moving average': { enabled: true, period: 20 },
      'bollinger bands': { enabled: true, period: 20, stdDev: 2 },
      rsi: { enabled: true, period: 14 },
    },
  },
}

// Array form, just the keys to enable.
plotOptions: { stockChart: { indicators: ['moving average', 'rsi'] } }
```

**2. At runtime** with `updateIndicator` / `setIndicatorParams` /
`removeIndicator`:

```js
stock.updateIndicator('bollinger bands');       // toggle on
stock.updateIndicator('rsi');                    // adds an RSI pane
stock.updateIndicator('macd');                   // RSI stays; MACD pane added below it
stock.removeIndicator('rsi');

stock.updateIndicator('rsi', { period: 21 });    // set params + ensure active
stock.setIndicatorParams('macd', { fastPeriod: 8, slowPeriod: 21 });
```

- **`updateIndicator(key)`** (one argument) **toggles** the indicator on/off.
- **`updateIndicator(key, params)`** and its explicit alias
  **`setIndicatorParams(key, params)`** set the params and ensure the
  indicator is active, they **never toggle it off**: an inactive indicator is
  added with the params, an active one is recomputed in place (data and
  streaming state), preserving zoom. This makes overlay periods (e.g. moving
  average) configurable programmatically.
- **`removeIndicator(key)`** removes explicitly.

## Overlays (price chart, multiple allowed)

| Key | Params | Notes |
|---|---|---|
| `"moving average"` | `period` | Simple moving average line. |
| `"exponential moving average"` | `period` | EMA line. |
| `"vwap"` | `source` | Volume-weighted average price, cumulative from the first bar. `source: "hlc3"` (typical price `(h+l+c)/3`, default) or `"close"`. Volume-less bars contribute 0. |
| `"bollinger bands"` | `period`, `stdDev` | Upper / middle / lower bands. |
| `"donchian channels"` | `period` (default 20) | Highest-high / lowest-low band over a trailing window. |
| `"keltner channels"` | `emaPeriod` (20), `atrPeriod` (10), `multiplier` (2) | EMA midline offset by `multiplier * ATR`. |
| `"fibonacci retracements"` | none | 0 / 23.6 / 38.2 / 50 / 61.8 / 100% levels. |
| `"linear regression"` | `period` | Regression trend line. |
| `"ichimoku cloud indicator"` | none | Full Ichimoku system (cloud + lines). |

## Oscillators (own pane, multiple at once)

Since 0.4.0 oscillators are **independent toggles**, like overlays: each active
oscillator gets its own pane, the panes stack below the price chart and share
the indicator area evenly (auto-resized on add/remove). Adding one never evicts
another. (On ≤ 0.3.x only one oscillator was active at a time.)

| Key | Notes |
|---|---|
| `"rsi"` | Relative Strength Index (0-100). `period` (default 14). |
| `"macd"` | MACD with signal line + histogram. `fastPeriod`, `slowPeriod`, `signalPeriod`. |
| `"volumes"` | Volume bars (needs `v` on the data). |
| `"price volume trend"` | PVT cumulative (needs `v`). |
| `"stochastic oscillator"` | %K and %D lines. |
| `"standard deviation indicator"` | Price volatility. |
| `"average directional index"` | ADX trend strength. |
| `"atr"` | Average True Range (Wilder-smoothed true range). `period` (default 14). |
| `"chaikin oscillator"` | Volume momentum (needs `v`). |
| `"commodity channel index"` | CCI overbought/oversold. |
| `"trend strength index"` | TSI momentum. |
| `"accelerator oscillator"` | Acceleration of price movement. |
| `"bollinger bands %b"` | Position within the bands (0-1). |
| `"bollinger bands width"` | Band width (volatility). |

## Analysis panes (0.5.0)

The dropdown groups these under an **Analysis** heading (a new optional `group`
on a registry entry), keeping analysis views distinct from technical indicators
without adding a second control. They are ordinary registry entries, so they
toggle, stack, zoom, theme, and persist exactly like an oscillator.

| Key | Notes |
|---|---|
| `"drawdown"` | Percent below the running peak, with zero pinned to the **top** of the axis (a drawdown is never positive) and the deepest point labelled on the pane. Measures on the chart's `analysis.drawdownBasis`, so it cannot disagree with `getRangeStats()` or `getDrawdown()`. Streams incrementally on `appendData` (the state is the running peak, so the step is O(1) with no warm-up). Takes **1.4** height shares by default. See [analysis.md](analysis.md). |

## Per-pane heights (0.5.0)

Panes divide the indicator area in proportion to a relative `heightRatio`
rather than always evenly: two panes at 1 and 2 split it one-third /
two-thirds. Most indicators take 1; the drawdown pane takes 1.4.

```js
new ApexStock(el, { panes: { drawdown: { heightRatio: 2 } } });   // at construction

stock.setPaneHeightRatio('rsi', 1.5);     // at runtime
stock.setPaneHeightRatio('rsi', null);    // restore that pane's default
stock.getPaneHeightRatios();
```

The heights always add up to the container exactly, and the ratios you set are
captured by `getState()` under a `panes` key.

## Key rules

- **Keys are lowercase full phrases**, `"moving average"`, not `"ma"`;
  `"bollinger bands"`, not `"bb"`. Unknown keys are no-ops. Lookup is
  case-insensitive.
- **Labels vs keys.** Display labels are title-cased per word ("Bollinger
  Bands", "Stochastic Oscillator") in the dropdown and in
  `listIndicators()` / `getIndicator()` `label` fields; the **keys and series
  names are unchanged** (always use the lowercase key in code).
- **Overlays and oscillators both stack** (0.4.0). Every active indicator
  round-trips through `getState()` / `setState()`, including multiple
  oscillators.
- **Panes share the area by ratio, not evenly** (0.5.0). See "Per-pane heights"
  above.
- **Every active indicator's current value is readable programmatically**
  (0.5.0) with `getDataAt(index)`, across both main-chart overlays and
  oscillator panes: `[{ name, value, color, pane, key? }]`. Values are plain
  numbers; warm-up entries are omitted. See [analysis.md](analysis.md).
- Streaming keeps indicators exact: on `appendData`, running indicators
  (including VWAP, ATR, Donchian, and Keltner) step forward from committed
  state (see [streaming-and-aggregation.md](streaming-and-aggregation.md)).

## Introspection: `listIndicators()` / `getIndicator(key)`

Enumerate every available indicator (built-in + custom) with metadata and live
state, for building custom pickers and settings panels:

```js
stock.listIndicators();
// [{ key: 'rsi', label: 'RSI', type: 'oscillator', kind: 'oscillator',
//    builtin: true, streamable: true, active: false, params: { period: 14 } }, ...]

stock.getIndicator('rsi');   // one entry (case-insensitive), or null if unknown
```

Entry fields: `key`, `label` (display name), `type`
(`'overlay'` / `'oscillator'`), `kind` (raw registry kind), `builtin`,
`active` (on this instance), `streamable` (has a live `appendData()` twin),
and `params` (current configurable params, `{}` when none).

## Custom indicators: `ApexStock.registerIndicator(name, def)`

Register your own indicator globally (before creating instances). It then works
with `updateIndicator(key)`, appears in the dropdown and `listIndicators()`,
and round-trips through `getState()` / `setState()`.

```js
ApexStock.registerIndicator('hh channel', {
  type: 'overlay',                      // or 'oscillator' (own pane)
  defaultParams: { period: 20 },
  calc(series, params) {
    // return ONE aligned line (number|null)[], a named multi-line map
    // ({ Upper: [...], Lower: [...] }), or ready-made ApexCharts series.
    return series.map((_, i) =>
      i + 1 < params.period ? null
        : Math.max(...series.slice(i + 1 - params.period, i + 1).map((b) => b.y[1])));
  },
  colors: ['#00E396'],
});

stock.updateIndicator('hh channel');
```

Definition fields: `type` + `calc` (required for the declarative form),
`defaultParams`, `color`/`colors`, `label` (defaults to a title-cased key),
`chartType` / `yaxis` / `chartOptions` (oscillator pane appearance),
`overwrite` (allow replacing a registered key), and an optional `stream` twin
(`{ seed, step, render }`) for incremental `appendData()` updates. Without a
`stream` twin, a custom indicator is recomputed from the full series on each
append (correct, but O(n)). An advanced form (`{ kind, build/apply/remove }`)
plugs a raw registry entry in verbatim.

## Pure calculation helpers

If you want the raw indicator values (e.g. for your own annotations or logic),
the instance exposes `calculate*` methods that return arrays without drawing
anything:

```js
const ma  = stock.calculateMovingAverage(series, period);
const ema = stock.calculateEMA(series, period);
const rsi = stock.calculateRSI(series, period);
const macd = stock.calculateMACD(series, fastPeriod, slowPeriod, signalPeriod);
const bb  = stock.calculateBollingerBands(series, period, stdDev);   // { middle, upper, lower }
const stoch = stock.calculateStochastic(series, period, smoothPeriod); // { k, d }
const adx = stock.calculateADX(series, period);
const cci = stock.calculateCCI(series, period);
const ichimoku = stock.calculateIchimoku(series); // { tenkan, kijun, senkouA, senkouB, chikou }
```

Indicator points are `{ x, y: number | null }`; `y` is `null` during the
warm-up period before the indicator has enough bars.
