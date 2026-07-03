# Technical Indicators

ApexStock ships 20+ built-in indicators, split into **overlays** (drawn on the
price chart) and **oscillators** (rendered in their own pane below the chart).

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

**2. At runtime** with `updateIndicator(key)` / `removeIndicator(key)`:

```js
stock.updateIndicator('bollinger bands');  // add / refresh (preserves zoom)
stock.updateIndicator('rsi');              // switches the active oscillator to RSI
stock.removeIndicator('rsi');
```

`updateIndicator` **toggles** an already-active indicator off if called again on
it, so use it to add or refresh, and `removeIndicator` to explicitly remove.

## Overlays (price chart, multiple allowed)

| Key | Params | Notes |
|---|---|---|
| `"moving average"` | `period` | Simple moving average line. |
| `"exponential moving average"` | `period` | EMA line. |
| `"bollinger bands"` | `period`, `stdDev` | Upper / middle / lower bands. |
| `"fibonacci retracements"` | none | 0 / 23.6 / 38.2 / 50 / 61.8 / 100% levels. |
| `"linear regression"` | `period` | Regression trend line. |
| `"ichimoku cloud indicator"` | none | Full Ichimoku system (cloud + lines). |

## Oscillators (own pane, one active at a time)

Adding a second oscillator **replaces** the first.

| Key | Notes |
|---|---|
| `"rsi"` | Relative Strength Index (0-100). |
| `"macd"` | MACD with signal line + histogram. |
| `"volumes"` | Volume bars (needs `v` on the data). |
| `"price volume trend"` | PVT cumulative (needs `v`). |
| `"stochastic oscillator"` | %K and %D lines. |
| `"standard deviation indicator"` | Price volatility. |
| `"average directional index"` | ADX trend strength. |
| `"chaikin oscillator"` | Volume momentum (needs `v`). |
| `"commodity channel index"` | CCI overbought/oversold. |
| `"trend strength index"` | TSI momentum. |
| `"accelerator oscillator"` | Acceleration of price movement. |
| `"bollinger bands %b"` | Position within the bands (0-1). |
| `"bollinger bands width"` | Band width (volatility). |

## Key rules

- **Keys are lowercase full phrases**, `"moving average"`, not `"ma"`;
  `"bollinger bands"`, not `"bb"`. Unknown keys are no-ops.
- **Overlays stack; oscillators don't.** Enable several overlays at once, but
  only the last-enabled oscillator is active.
- Streaming keeps indicators exact: on `appendData`, running indicators step
  forward from committed state (see
  [streaming-and-aggregation.md](streaming-and-aggregation.md)).

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
