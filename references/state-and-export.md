# State Persistence, Export & Cross-Chart Sync: ApexStock

`getState()` / `setState()`, the unified `export()` API (including PDF), and
`ApexStock.sync()`.

---

## State

`getState()` returns a schema-versioned, `JSON.stringify`-safe snapshot.
`setState(state)` restores it, **after `render()`**. Older states migrate
forward automatically; `ApexStock.migrateState(state)` migrates up front.

```js
localStorage.setItem('chart', JSON.stringify(stock.getState()));

// later, in a new session
stock.render();
stock.setState(JSON.parse(localStorage.getItem('chart')));
```

`ApexStock.STATE_VERSION === 2`.

### The v2 shape

```ts
type ApexStockState = {
  version: number;
  theme: { mode: 'light' | 'dark'; preset?: string | null };
  chartType: string;              // 'candlestick' | 'heikinashi' | 'renko' | 'line' | 'area' | 'ohlc'
  indicators: IndicatorState[];   // active indicators + params, in application order
  drawings: object[];             // v2+. Plain-JSON geometry + style; restored verbatim.
  eventMarkers: object[];         // v2+. { x, type, label?, color?, glyph?, position?, meta? }
  annotations: object[];          // v2+. y/x lines, bands, points, text
  priceLines: object[];           // v2+. Declarative config ONLY (see below)
  priceScale: { mode, base, logBase, indexBase } | null;   // v2+. null = default linear
  panes: Record<string, { heightRatio: number }> | null;   // v2+. null = every pane at default
  comparison: ComparisonState | null;                      // v2+. Setup only (see below)
  zoom: { minX: number; maxX: number } | null;             // null = full/auto
};
```

`theme.preset` is the active named preset, or `null` for a plain mode.

`panes` holds only the height ratios a consumer **set**. Which panes *exist* is
derived from `indicators`, so it is not duplicated here.

### What is NOT captured, and how to restore it

Two things, both for the same reason: they are not serializable, or they are
not the chart's to own.

**1. Price-line interactive callbacks.** `onCross`, `onMove`, and `onRemove`
are functions. The declarative config round-trips; the callbacks do not. Re-bind
them after `setState`:

```js
stock.setState(saved);
for (const line of stock.getPriceLines()) {
  if (line.kind === 'alert') stock.updatePriceLine(line.id, { onCross: notify });
}
```

**2. Comparison instrument data.** Your app fetches it, it runs to thousands of
bars per instrument, and it would be stale the moment it was written to
storage. So `setState` restores the **setup** (mode, benchmark, alignment
policy, and each instrument's name and color), keeps any instrument whose data
is still loaded, and emits `comparisonRestoreNeeded` for the rest:

```js
stock.on('comparisonRestoreNeeded', async ({ names }) => {
  for (const name of names) {
    stock.addComparison({ name, data: await fetchBars(name) });  // remembered color reapplied
  }
});
stock.setState(saved);
```

A benchmark whose instrument has not come back yet is remembered by name, with
the primary filling the role until it does. Same division of labour as the price
lines' callbacks.

**Measurements need no key of their own**: a measurement *is* a `measure`
drawing, so it already round-trips inside `drawings`.

### Custom drawing types

A serialized custom drawing re-renders only if
`ApexStock.registerDrawingTool(name, def)` has been called again in the new
session **before** `setState()`.

---

## Export

`export({ format })` is the unified entry point over `png` / `svg` / `pdf` /
`csv` / `json`, returning a consistent
`Promise<{ format, blob, url, text? }>`. It folds in `exportImage` and
`exportData`, both of which remain available and unchanged.

```js
const { blob, url } = await stock.export({ format: 'png', scale: 2 });
const { text }      = await stock.export({ format: 'csv', include: ['indicators'] });
await stock.export({ format: 'pdf', include: ['analysis'], range: 'visible', download: true });
```

| Option | Applies to | Meaning |
|---|---|---|
| `format` | all | `'png'` (default) / `'svg'` / `'pdf'` / `'csv'` / `'json'` |
| `scale` | png | Resolution multiplier. |
| `range` | csv, json, pdf | `'all'` (default) or `'visible'`. |
| `include` | csv, json, pdf | Extra content: `'ohlc'` (always present), `'indicators'`, `'analysis'`. |
| `summary` | pdf | `string[]` overriding the generated summary band. |
| `includeVolume` | csv, json | Force the volume column on/off (defaults to on when any point has a `v`). |
| `raw` | csv, json | Emit raw `x` instead of ISO time. |
| `pretty` | json | Pretty-print (default true). |
| `download` / `filename` | all | Also trigger a browser download (extension added). |

### `include`: carrying the analysis into an export

`include` means something appropriate to each medium.

**For `csv` / `json`:**

- `'indicators'` adds **one column per active indicator series**, main-chart
  overlays *and* oscillator panes, with `null` through each one's warm-up.
- `'analysis'` adds `return` (percent change from the previous bar) and
  `drawdown` (percent below the running peak, per `analysis.drawdownBasis`).

**For `pdf`:**

- `'analysis'` sets a text summary below the chart image: the window's dates
  and bar count, change, high, low, average, volume, annualized return,
  volatility, max drawdown, and the comparison leaderboard when one is active.
  Override it wholesale with `summary: string[]`.

Indicator columns come from the live chart state and analysis columns from the
same engine `getRangeStats` uses, so **an exported number matches an on-chart
one**.

### Rules

- **The OHLC spine is always present**, so a CSV keeps round-tripping through
  `ApexStock.fromCSV` whatever you add. An extra column whose name collides
  with a spine column is **suffixed**, not allowed to overwrite it.
- **Range statistics are not columns.** They are a summary, not a per-bar
  value; read them from `getRangeStats()`.
- **The PDF summary follows `range`**, defaulting to the visible window, so the
  exported numbers describe what you were looking at.
- **PDF is dependency-free.** `buildPdfFromJpeg` grows the page for a text band
  below the image and sets it in Helvetica, one of the PDF base-14 fonts, so
  there is nothing to embed.
- **PNG can fall back to SVG.** It composites the price chart and every
  oscillator pane into one raster; browsers that block `<foreignObject>`
  rasterization get SVG instead, flagged `fallback: true` on the result. SVG is
  always available.

Base columns are `time, open, high, low, close[, volume]`, with `time` as
ISO-8601 for numeric timestamps.

---

## `ApexStock.sync(instances, options?)`

Links independent ApexStock instances so panning or zooming one mirrors the
visible range to the others, and a crosshair on one draws a vertical guide at
the same x on the others.

```js
const link = ApexStock.sync([priceChart, volumeChart], { zoom: true, crosshair: true });
// ...
link.disconnect();
```

| Option | Default | Meaning |
|---|---|---|
| `zoom` | `true` | Mirror the visible range. |
| `crosshair` | `true` | Draw a guide at the same x on the other charts. |
| `crosshairColor` | theme-derived | Guide color. |

Built on the public event bus plus `setVisibleRange()`, **not** ApexCharts'
native `group`, with echo suppression so there is no feedback loop. The
crosshair guide is a lightweight per-chart DOM overlay positioned from each
chart's own axis.

Returns a handle with `disconnect()`, which unlinks the instances and removes
all guides and listeners. Idempotent.

---

## Pitfalls

1. **Calling `setState()` before `render()`.** Restore after the chart exists.
2. **Expecting price-line callbacks to survive a round trip.** They are
   functions. Re-bind with `updatePriceLine(id, { onCross })`.
3. **Expecting comparison data to survive a round trip.** Only the setup is
   captured. Subscribe to `comparisonRestoreNeeded` and hand the bars back.
4. **Restoring a custom drawing type without re-registering it.** Call
   `ApexStock.registerDrawingTool` before `setState`.
5. **Looking for a `measurements` key.** Measurements ride in `drawings`.
6. **Assuming the 0.4.0 docs' persistence boundary.** Annotations and price
   lines *are* captured now; what is not captured is narrower and more specific
   than "annotations and price lines".
7. **Expecting a PDF summary without asking for it.** Pass
   `include: ['analysis']`.
8. **Naming an extra export column after a spine column.** It gets suffixed
   rather than replacing OHLC, which is what keeps `fromCSV` working.
