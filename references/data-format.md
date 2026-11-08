# Data Format: OHLC Points, Chart Types, Adapters, Export

ApexStock reads its financial data from **`series[0].data`**. Only the first
series holds the OHLC candles: extra `series` entries are treated as
indicators, not instruments. To overlay another symbol, use comparison mode
(`addComparison`, see [trading-overlays.md](trading-overlays.md)).

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
  keys, that shape is not recognized and the point is dropped (run such rows
  through `ApexStock.normalize` instead, see below).
- **`x` types.** Epoch-ms `number`, an ISO date string (`'2024-01-01'`), or a
  `Date`. Numeric timestamps produce a true numeric/datetime x-axis (correct
  zoom/scroll); string categories produce a category axis.
- **Sort ascending by `x`.** Streaming appends assume the newest bar is last.
- **`v` (volume) is optional** and only needed for volume-based indicators
  (`"volumes"`, `"price volume trend"`, `"chaikin oscillator"`, `"vwap"`;
  VWAP treats volume-less bars as contributing 0).
- **Normalization.** ApexStock validates and normalizes at the constructor and
  at `update()` / `appendData()` boundaries: points missing a valid `x` or a
  4-number `y` are dropped, and out-of-order points are handled. A "missing"
  candle almost always means a malformed `y`.
- **The close (`y[3]`) drives most indicators.**

## Data adapters (`normalize` / `fromArrays` / `fromCSV`)

Real-world data rarely arrives in the `{ x, y: [o,h,l,c], v? }` shape. Three
**static** helpers convert the common shapes; their output is validated and
time-sorted (same cleaning as the render pipeline), so pass it straight into
`series`:

```js
// 1) Arrays of objects or tuples. Columns match by case-insensitive alias:
//    date/time -> x, o -> open, c -> close, vol -> volume, ...
const series = ApexStock.normalize([
  { date: '2024-01-01', o: 100, h: 110, l: 95, c: 105, v: 1_000_000 },
]);
ApexStock.normalize(rows, { x: 'Date', close: 'Adj Close' });  // mapping wins over aliases
ApexStock.normalize([[1704067200000, 100, 110, 95, 105, 1_000_000]]); // tuples: [x,o,h,l,c,v]

// 2) Parallel column arrays. Only `close` is required; missing OHLC columns
//    are derived from it (a close-only feed becomes flat candles).
ApexStock.fromArrays({ t: [t0, t1], open: [100, 105], high: [110, 115],
                       low: [95, 100], close: [105, 112] });
ApexStock.fromArrays({ close: [105, 112, 116] });   // x falls back to 0,1,2

// 3) CSV text. The header row drives alias resolution by default.
ApexStock.fromCSV('Date,Open,High,Low,Close,Volume\n2024-01-01,100,110,95,105,1000000');
ApexStock.fromCSV(csvText, { header: false });                    // positional columns
ApexStock.fromCSV(csvText, { delimiter: ';', mapping: { close: 'Last' } });
```

`fromCSV` handles quoted fields, embedded delimiters, headerless data, custom
delimiters, and CRLF/LF line endings.

## Exporting data and images

```js
// Data: CSV or JSON of the OHLC series.
const csv = stock.exportData({ format: 'csv' });                        // all bars
stock.exportData({ format: 'json', range: 'visible', download: true }); // only the zoomed window
```

`exportData({ format, range?, includeVolume?, raw?, pretty?, download?,
filename? })` returns the serialized string. `format` is `'csv'` (default) or
`'json'`; `range` is `'all'` (default) or `'visible'`. Columns are
`time,open,high,low,close[,volume]`, with ISO-8601 time for numeric timestamps
(`raw: true` keeps the raw value). The CSV round-trips through
`ApexStock.fromCSV`.

```js
// Image: real PNG (main chart + oscillator panes composited) or vector SVG.
const { blob, url, fallback } = await stock.exportImage({ format: 'png', scale: 2 });
await stock.exportImage({ format: 'svg', download: true });
```

`exportImage({ format, scale?, download?, filename? })` returns
`Promise<{ format, blob, url, fallback? }>`. Browsers that block rasterizing
fall back to SVG and set `fallback: true`. The toolbar download button uses the
same path.

## Chart types

ApexStock supports candlestick, line, area, column, heikin-ashi, and renko.
The chart-type switch in the toolbar changes the rendering of the same OHLC
data; you generally don't hand-configure the ApexCharts `chart.type`, feed
OHLC data and let ApexStock manage the financial rendering. Line/area views
use the close. The active type is captured by `getState()`.

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
**ApexStock 0.6.0 (like 0.5.2) targets ApexCharts 7.1+ and 8.x (peer
dependency `apexcharts@^7.1.0 || ^8.0.0`).** 0.5.0 and 0.5.1 accepted `^7.1.0`
only, and 0.4.0 required `^6.7.0`. The 0.6.0 Ignite UI add-on needs ApexCharts
8.8.0 or later. No extra ApexCharts import is needed on 8.x: every
chart type ApexStock draws stays in the default bundle.

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
