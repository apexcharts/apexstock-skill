# Data Format: OHLC Points, Chart Types, Timestamps

ApexStock reads its financial data from **`series[0].data`**. Only the first
series holds the OHLC candles.

## The OHLC point

```ts
type OHLCPoint = {
  x: number | string | Date;            // timestamp for the candle
  y: [number, number, number, number];  // [open, high, low, close]
  v?: number;                            // optional volume
};

type Series = OHLCPoint[];
```

```js
const candles = [
  { x: '2024-01-01', y: [100, 110, 95, 105], v: 1_000_000 },
  { x: '2024-01-02', y: [105, 115, 100, 112], v: 1_200_000 },
  { x: '2024-01-03', y: [112, 118, 108, 109], v: 900_000 },
];

const stock = new ApexStock(el, {
  chart: { height: 600 },
  series: [{ name: 'AAPL', data: candles }],
});
stock.render();
```

### Rules

- **Four prices, one array.** The prices live in a single `y` tuple ordered
  `[open, high, low, close]`. Do **not** use separate `o` / `h` / `l` / `c`
  keys, that shape is not recognized and the point is dropped.
- **`x` types.** Epoch-ms `number`, an ISO date string (`'2024-01-01'`), or a
  `Date`. Numeric timestamps produce a true numeric/datetime x-axis (correct
  zoom/scroll); string categories produce a category axis.
- **Sort ascending by `x`.** Streaming appends assume the newest bar is last.
- **`v` (volume) is optional** and only needed for volume-based indicators
  (`"volumes"`, `"price volume trend"`, `"chaikin oscillator"`).
- **Normalization.** ApexStock validates and normalizes at the constructor and
  at `update()` / `appendData()` boundaries: points missing a valid `x` or a
  4-number `y` are dropped, and out-of-order points are handled. A "missing"
  candle almost always means a malformed `y`.
- **The close (`y[3]`) drives most indicators.**

## Chart types

ApexStock supports candlestick, OHLC bars, line, area, and heikin-ashi. The
chart-type switch in the toolbar changes the rendering of the same OHLC data;
you generally don't hand-configure the ApexCharts `chart.type`, feed OHLC data
and let ApexStock manage the financial rendering. Line/area views use the close.

## Options object shape

```ts
type StockChartOptions = {
  chart: any;                                   // standard ApexCharts chart config
  series: Array<{ name?: string; data: OHLCPoint[] }>;
  theme?: { mode?: 'light' | 'dark' };
  plotOptions?: { stockChart?: { indicators?: /* see indicators.md */ } };
};
```

`StockChartOptions` is a **superset of standard ApexCharts options**: anything
you'd put under `chart` (height, id, zoom, animations, events, …) is respected.

> **ApexStock mutates the options object** it receives, it sets `chart.id`,
> writes back the normalized `series`, and strips a nullish top-level `theme`.
> Pass a plain object you don't mind being touched (the framework wrappers hand
> it a shallow copy of `chart` and each series entry).

## Replacing data vs. appending

- **Full replace / reconfigure:** `stock.update({ series: [{ data: next }] })`
  merges options and re-renders, preserving zoom, active indicators, theme,
  and chart type.
- **Live data:** `stock.appendData(barOrBars, opts)` is incremental, the fast
  path for streaming. See [streaming-and-aggregation.md](streaming-and-aggregation.md).
