# Analysis Reference: ApexStock

Everything added by the 0.5.0 financial-analysis workspace: window statistics,
drawdown, range measurements and the analysis panel, multi-instrument
comparison, the crosshair data readout, and price-scale modes.

Four surfaces sit over **one** statistics engine, so a pane, a panel, an export
column, and a returned object can never disagree about the same number.

## Contracts that hold everywhere in this file

1. **Values are unrounded.** The consumer formats. `stats.change.percent` is
   `20.4183...`, not `20.42`.
2. **Every percent-like value is in percent units.** `20.42` means +20.42%. Do
   not multiply by 100.
3. **Anything the data cannot support is `null`, with the reason in
   `warnings`.** This is the contract, not an error. An annualized return is
   omitted for spans under `minAnnualizeDays` (default 30) rather than
   extrapolated; an annualized volatility is omitted for intraday bars unless
   `periodsPerYear` is supplied.
4. **Counts are bars, not trading sessions.** ApexStock owns no market
   calendar, so it will not claim to know what a session is.

## Construction options

```js
new ApexStock(el, {
  // ...
  analysis: {
    source: 'close',          // 'close' | 'open' | 'high' | 'low'
                              // what the anchors, the change, and the averages read
    drawdownBasis: 'close',   // 'intrabar' measures each bar's LOW against the running
                              // high: the conservative figure
    periodsPerYear: undefined,// annualization convention; required to annualize intraday
    minAnnualizeDays: 30,     // below this span, no annualized return at all
    by: 'auto',               // how a bare number endpoint is read: 'index' | 'x' | 'auto'
    measure: { snap: 'close', label: (stats, { selection, drawing }) => [...] },
    panel: true,              // or an options object, or false for headless use
    comparison: { /* see "Comparison" below */ },
  },
  panes: { drawdown: { heightRatio: 2 } },
});
```

Every option is overridable per call.

---

## `getRangeStats(from, to, opts?)`

Everything about a selected region. Endpoints may be given **in either order**
as a bar index, an epoch-ms `x`, a `Date`, or a date string; `analysis.by`
decides how a bare number is read.

```js
const stats = stock.getRangeStats('2024-01-02', '2024-03-15');
```

| Field | Meaning |
|---|---|
| `from` / `to` | `{ index, x, value }`, each resolved to a bar that actually exists. |
| `change` | `{ absolute, percent }` over the spanned bars, measured on `analysis.source`. |
| `bars` | Inclusive bar count. Bars, not trading sessions. |
| `upBars` / `downBars` / `flatBars` | The split. |
| `spanMs` / `calendarDays` | Wall-clock span. |
| `annualized` | `{ return, basis: 'calendar' }`, or `null` under `minAnnualizeDays`. |
| `high` / `low` | `{ value, index, x }`: the **true** extremes in the region, not the endpoint closes. |
| `average` | `{ close, volume }`. |
| `total` | `{ volume }`. |
| `volatility` | Sample standard deviation of log returns, **annualized**. `null` on intraday bars with no `periodsPerYear`. |
| `drawdown` | The deepest drawdown **within** the region, with its peak, trough, recovery, and three separate durations: `barsToTrough` (the decline), `barsToRecovery` (the recovery), and `barsUnderwater` (the whole episode). |
| `basis` | `{ source, drawdown }`: what was actually measured, echoed back. |
| `warnings` | Why anything is `null`. |

Three durations rather than one, because a decline and a recovery are different
questions and a single "duration" would answer neither.

---

## `getDrawdown(opts?)` and the drawdown pane

`getDrawdown()` returns per-bar drawdown as a **plottable series**, plus every
drawdown episode, the deepest, and the current one.

```js
const dd = stock.getDrawdown();                    // uses analysis.drawdownBasis
const conservative = stock.getDrawdown({ basis: 'intrabar' });
```

`basis: 'intrabar'` measures each bar's low against the running high; `'close'`
measures close against close. An explicit per-call `basis` wins over the
chart's `analysis.drawdownBasis`.

**The pane.** `updateIndicator('drawdown')` toggles an analysis pane plotting
percent below the running peak, with zero pinned to the **top** of the axis (a
drawdown is never positive) and the deepest point labelled on the pane itself.

```js
stock.updateIndicator('drawdown');
```

It is an ordinary oscillator-registry entry, so it inherits the whole pane
machinery with no new API: stacking with other panes, the shared zoom, the
theme rebuild, `getDataAt`, `getState` / `setState`, and the settings-free
dropdown entry. It measures on the chart's `analysis.drawdownBasis`, so the
pane, `getRangeStats()`, and `getDrawdown()` cannot disagree, and the basis
round-trips in state with the pane.

It **streams incrementally on `appendData`**: the state is the running peak, so
the step is O(1) with no warm-up period. The pane's "max" label is re-asserted
after each append, so a new deeper trough moves it instead of leaving a stale
line behind.

The indicators dropdown groups it under an **Analysis** heading (a new optional
`group` on a registry entry), keeping analysis views distinct from technical
indicators without adding a second control.

### Per-pane heights

Panes divide the indicator area in proportion to a relative `heightRatio`
rather than always evenly. The drawdown pane takes **1.4** shares (it is
cumulative, so it earns the room); everything else takes 1.

```js
new ApexStock(el, { panes: { drawdown: { heightRatio: 2 } } });   // at construction

stock.setPaneHeightRatio('drawdown', 2);   // at runtime
stock.setPaneHeightRatio('drawdown', null);// restore this pane's default
stock.getPaneHeightRatios();               // { drawdown: 2, rsi: 1 }
```

The heights always add up to the container exactly. Ratios you set are captured
by `getState()` under the `panes` key; which panes *exist* is derived from
`indicators`, so only the layout is stored.

---

## Measurements and the analysis panel

The `measure` drawing tool reports what happened over the region it spans, not
just the distance between its two anchors.

```js
const { id, stats } = stock.measureRange('2024-01-02', '2024-03-15');
stock.getMeasurements();      // MeasurementInfo[]
stock.getMeasurement(id);
stock.clearMeasurement(id);   // omit id to clear all; returns the count removed
```

A measurement **is** a real `measure` drawing: it renders, can be selected and
dragged, stays anchored to its bars through zoom and pan, and round-trips
through `getState()` / `setState()` with the other drawings. **There is no
separate state key for measurements.**

### `change` vs `selection`, and why there are two

A measurement reports **two** changes, separately named, because they answer
different questions:

- **`stats.change`** is the instrument's close-to-close move over the spanned
  bars. This is the figure every other statistic is consistent with.
- **`selection`** is the delta between the two anchors the user dragged. This
  is what the box's height shows, and the right number for a low-to-high swing.

They agree when the anchors sit on the closes. `analysis: { measure: { snap } }`
pulls hand-drawn anchors onto the bar values (`true` means the close), which
makes them agree by construction.

### The analysis panel

An on-chart readout of the region statistics: change, duration, true high and
low, average price and volume, volatility, annualized return, max drawdown, and
the decline and recovery durations.

Automatic by default: it appears while a measurement exists and hides when the
last one is cleared.

```js
new ApexStock(el, {
  analysis: {
    panel: {
      position: 'top-right',            // 'top-left' | 'top-right' | 'bottom-left' | 'bottom-right'
      metrics: ['change', 'high', 'low'],// see AnalysisPanel.METRICS for the default set
      formatters: { price, percent, volume, date },
      title: 'Selection',
      placeholder: 'Drag to measure',
    },
  },
});

stock.showAnalysisPanel();
stock.hideAnalysisPanel();
stock.isAnalysisPanelVisible();
```

`analysis: { panel: false }` opts out entirely for headless use. Every row
carries a `data-metric` attribute for styling.

The on-chart measure label is multi-line and comes from the same engine;
`analysis.measure.label(stats, { selection, drawing })` replaces it wholesale.

### Events

| Event | Payload |
|---|---|
| `rangeMeasured` | `{ id, from, to, selection, stats, source }`, `source` is `'drag'` / `'api'` / `'coreRuler'`. Fires once per **settled** change: one drag is one event, and a plain zoom or pan produces none. |
| `measurementRemoved` | `{ id }` |

### ApexCharts measure-ruler interop

When you load the optional `apexcharts/features/measure` module (**not in the
default `apexcharts` bundle in 7.x or 8.x, so add
`import 'apexcharts/features/measure'`; ApexCharts 8's `apexcharts/full` bundle
already includes it**) and set
`chart: { measure: { enabled: true } }`, ApexStock supplies `chart.measure.label`
from the same statistics engine and re-emits the core `measured` event as
`rangeMeasured` with `source: 'coreRuler'`, so both gestures produce one
consistent readout.

ApexStock keeps its own measure drawing as the **primary** gesture: the core
ruler's pins live on the chart instance rather than in ApexStock's state, and
its numbers are geometric only.

### Reshaping (0.5.0)

Selecting a measure box, trend line, ray or fib now puts a drag handle on
**each of its two anchors**: drag one to change what it covers from that end,
and the statistics recompute over the bars it now spans. Previously a
measurement could only be translated, so its span was frozen at whatever
created it and re-measuring meant deleting and redrawing.

Dragging the **body** still translates with the span intact, which is
deliberate: that is how a measured move is projected onto a later breakout. A
`locked: true` drawing shows no handles.

---

## Comparison

Comparison mode answers "who's up more" fairly even when the instruments do not
share a calendar.

```js
stock.addComparison({ name: 'MSFT', data: msftBars });
stock.setComparisonBenchmark('MSFT');
stock.setComparisonMode('relative');
const board = stock.getComparisonStats();
```

### Alignment happens first

Every instrument, **the primary included**, is aligned onto one shared x grid
before anything is normalized, so a newer listing, a different holiday
calendar, or a missing day no longer skews the comparison.

```js
stock.setComparisonOptions({
  join: 'union',        // 'union' (default) keeps every x any instrument has
                        // 'primary' resamples onto the primary's bars
                        // 'intersection' keeps only x values every instrument has
  fill: 'hold',         // 'hold' (default) carries the last observation
                        // 'gap' leaves it null
                        // 'drop' removes the x value for everyone
  resample: null,       // plot the filled grid instead of only real observations.
                        // Defaults to true for join: 'primary', which exists to resample.
});
```

A carried-forward value is used for the **math** but is **not plotted**, so a
line never shows a bar its instrument does not have.
`getComparisonStats()` reports how many points were filled. A point before an
instrument's first observation is always `null`, never backfilled.

### Baseline policies: where 0% sits

| `baseline` | Meaning |
|---|---|
| `'common'` | **The default since 0.5.0.** The first x where every instrument has data. Earlier history still plots, as a negative percent, rather than being hidden. |
| `'own'` | Each instrument's own first point. The pre-0.5.0 behavior, kept for compatibility. |
| `'visible'` | The left edge of the visible window; follows the zoom. |
| a number | An explicit x value. |

The default changed because with more than one instrument, or one whose history
starts before or after the primary's, `'own'` made each line start at 0% on a
different date, which quietly overstated the newest listing. A single
instrument on a matching calendar renders identically either way.

`'common'` can fall back to `'own'` when no x value has data for every
instrument. The `comparisonChange` event reports the policy actually
**applied**, so read `baseline` from there rather than assuming.

### The five modes

| Mode | Meaning |
|---|---|
| `'absolute'` | Raw values. |
| `'percent'` | Percent change from the baseline. The default. |
| `'indexed'` | The baseline reads `indexBase` (default 100): the "100 = starting value" view. |
| `'relative'` | `percentChange(asset) - percentChange(benchmark)`, in **percentage points**. Zero means "kept pace". |
| `'ratio'` | `asset / benchmark`, rebased to `indexBase` (set `rebaseRatio: false` for the raw quotient). |

### The benchmark is a role, not a ticker

```js
stock.setComparisonBenchmark('MSFT');          // any added instrument
stock.setComparisonBenchmark('__primary__');   // the chart's own symbol (the default)
stock.getComparisonBenchmark();
```

In `relative` and `ratio` mode the benchmark's own line becomes the flat
reference, so what everything is measured against is visible on the chart.
Removing that instrument hands the role back to the primary. **No symbol is
hard-coded anywhere in the library.**

### `getComparisonStats(opts?)`

The leaderboard, ready to render: one row per instrument (**the primary
included**, flagged `primary: true`) with change, excess return vs the
benchmark, high and low, volatility, worst drawdown, rank, and grid coverage.

```js
const rows = stock.getComparisonStats({ from, to });
```

The window runs from the baseline to the last observation, so with
`baseline: 'visible'` the rows follow the zoom. Volatility and drawdown are
measured on each instrument's **own** observations, so a weekly line on a daily
grid is not annualized as if it had 252 bars a year.

### Declarative defaults

```js
new ApexStock(el, {
  analysis: {
    comparison: { mode: 'indexed', benchmark: 'SPY', join: 'union',
                  fill: 'hold', baseline: 'common', indexBase: 100, source: 'close' },
  },
});
```

A benchmark named here is remembered until its instrument is added, with the
primary filling the role (and saying so in `warnings`) until then. Unknown
option values are warned about and ignored rather than silently changing what
the numbers mean.

### The `comparisonChange` event

```js
stock.on('comparisonChange', ({ reason, mode, benchmark, baseline, instruments, stats, warnings }) => {
  render(stats);   // the recomputed leaderboard
});
```

Skipped entirely when nothing is subscribed.

### Rules

- **A comparison instrument may not take the primary series' name.**
  `addComparison({ name })` refuses the name of `series[0]` and the reserved
  `'__primary__'`; previously that filtered the price series off the chart.
- **Rebasing preserves the zoom** (fixed in 0.5.0). Adding or removing an
  instrument, or switching mode, used to snap a zoomed chart back to its full
  history and desync the indicator panes. Worst under `baseline: 'visible'`,
  where a zoom is what triggers the rebase, so the gesture undid itself. The
  window is now restored after the rebuild, across the panes too, and only when
  the chart was genuinely zoomed, so an instrument whose history reaches
  further back than the primary's still widens the view instead of being
  clipped to it.

---

## `getDataAt(index?)`

A read-only, structured snapshot at a data-point index: OHLC, volume, change vs
the previous close, and **every active indicator's value across both the
main-chart overlays and the oscillator panes**.

```js
stock.on('crosshairMove', ({ dataPointIndex }) => {
  const readout = stock.getDataAt(dataPointIndex);
  // readout.indicators: [{ name, value, color, pane, key? }]
});
```

The programmatic complement to the on-chart legend and the `crosshairMove`
event: pass the event's `dataPointIndex` to build a custom legend, side panel,
or tooltip. Values are plain numbers (unformatted); unavailable ones are `null`
(volume, change) or omitted (warm-up). Defaults to the latest bar when the index
is omitted or out of range.

---

## Price-scale modes

Four modes for the primary price y-axis.

```js
new ApexStock(el, { priceScale: { mode: 'logarithmic', logBase: 10 } });

stock.setPriceScale('percent');                  // % change from the first bar
stock.setPriceScale('percent', { base: 120 });   // ...or an explicit base
stock.setPriceScale('indexed', { indexBase: 100 });
stock.getPriceScale();   // { mode, base, logBase, indexBase }
```

| Mode | Meaning |
|---|---|
| `'linear'` | The default. |
| `'logarithmic'` | Native log axis, optional `logBase`. |
| `'percent'` | Percent change from the first bar, or an explicit `base`. |
| `'indexed'` | Index where the baseline reads `indexBase` (default 100). |

**`percent` and `indexed` are affine relabelings of the axis**, so indicators,
drawings, annotations, and trading price lines stay in true price space and are
unaffected. Only `logarithmic` changes the gridline distribution.

The mode persists across theme and chart-type switches and `appendData` (the
baseline is recomputed from the current first bar), is captured by `getState()`
under `priceScale`, and emits a `priceScaleChange` event
(`{ mode, base, logBase, indexBase }`).

---

## `ApexStock.stats`: the engine, headless

The whole statistics engine with **no chart and no DOM**, for server-side
reports, tests, and workers.

```js
ApexStock.stats.rangeStats(series, '2024-01-02', '2024-03-15');
ApexStock.stats.drawdown(series);
ApexStock.stats.align({ AAPL, SPY }, { join: 'intersection' });
```

| Function | Purpose |
|---|---|
| `rangeStats(series, from, to, opts?)` | The same object `getRangeStats` returns. |
| `drawdown(series, opts?)` | The same object `getDrawdown` returns. |
| `returns` / `volatility` / `annualize` / `inferPeriodsPerYear` | The return and risk primitives. |
| `align(instruments, opts?)` | The multi-instrument primitive: a configurable x grid (`join`), a missing-data policy (`fill`), and a baseline policy (`'common'` by default). |
| `baseline` / `rebase` | `rebase` gives percent-change or "100 = starting value" columns. |
| `relative` | A benchmark spread or ratio, with the benchmark as a role rather than a hard-coded symbol. |

---

## Pitfalls

1. **Multiplying a percent by 100.** Values are already in percent units.
2. **Treating `null` as a failure.** It is the documented "the data cannot
   support this" answer; the reason is in `warnings`.
3. **Expecting an annualized figure on a short or intraday window.** Set
   `periodsPerYear` for intraday volatility, and expect no annualized return
   under `minAnnualizeDays` (default 30).
4. **Confusing `stats.change` with `selection` on a measurement.** `change` is
   the close-to-close move over the spanned bars; `selection` is the delta
   between the dragged anchors. Turn on `analysis.measure.snap` to make them
   agree.
5. **Looking for a `measurements` key in `getState()`.** There is none: a
   measurement is a `measure` drawing and rides in `drawings`.
6. **Assuming the old comparison baseline.** The default is `'common'` since
   0.5.0, not `'own'`.
7. **Reading `baseline` from your own config instead of the event.** The
   applied policy can fall back to `'own'`; `comparisonChange.baseline` reports
   what actually happened.
8. **Expecting a carried-forward comparison point to be plotted.** It is used
   for the math and left off the line, unless `resample: true`.
9. **Passing `basis` to `getDrawdown` and expecting the chart option to be
   ignored, or vice versa.** The chart-level option is `analysis.drawdownBasis`;
   the per-call option is `basis`. The per-call one wins. (Before 0.5.0's fix,
   `getDrawdown()` ignored the chart-level setting entirely.)
10. **Expecting range statistics as export columns.** They are a summary, not a
    per-bar value. `include: ['analysis']` gives per-bar `return` and
    `drawdown`; the window summary comes from `getRangeStats()`.
