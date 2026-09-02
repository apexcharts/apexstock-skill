# Trading Overlays: Price Lines, Annotations, Drawings, Comparison

Four distinct layers, pick the right one:

| Layer | API | Anchoring | Serialized by `getState()`? |
|---|---|---|---|
| Price lines | `addPriceLine` + typed shortcuts | y (price) | No (persist via `getPriceLines()`) |
| Annotations | `addAnnotation` | data coords (price/time) | No (persist via `getAnnotations()`) |
| Drawings | `addDrawing` + mouse toolbar | data coords, reproject on zoom/pan | **Yes** (schema v2) |
| Comparison | `addComparison` | secondary y-axis | No |

## Price lines

Horizontal price lines mark order / stop-loss / take-profit / alert levels. They
**persist** across zoom, theme change, chart-type switch, and streaming appends.

```js
const id = stock.addOrderLine({ price: 98.5, side: 'buy', label: 'Entry', draggable: true });
stock.addStopLoss({ price: 95 });
stock.addTakeProfit({ price: 104 });
stock.addAlert({ price: 100, onCross: (e) => notify(e.direction) });

stock.updatePriceLine(id, { price: 97 });  // reprice / relabel
stock.removePriceLine(id);                  // remove one
stock.clearPriceLines();                    // remove all
stock.getPriceLine(id);                     // -> a copy of the config, or null
stock.getPriceLines();                      // -> copies of all configs
```

`addPriceLine(config)` is the generic form; `addOrderLine` / `addStopLoss` /
`addTakeProfit` / `addAlert` are typed shortcuts with sensible default colors.
Each `add*` returns the line id (a string), or `null` on invalid input (e.g. a
missing `price`).

### `PriceLineConfig` fields

| Field | Description |
|---|---|
| `price` | **Required.** The y level for the line. |
| `id` | Stable id; auto-generated when omitted. |
| `side` | `"buy"` / `"sell"` for order lines (drives the default color). |
| `label` | Text shown on the line. |
| `color`, `textColor`, `strokeDashArray`, `width`, `labelPosition` | Appearance. Colors default from the themeable `colors.tradingOverlays` group. |
| `draggable` | Drag the line vertically to reprice; fires `onMove({ id, price })` on drop. |
| `closable` | Shows a ✕ button; clicking removes the line and fires `onRemove({ id })`. |
| `onCross` | Fired as `{ id, type, price, direction, bar }` when a closed bar (from `appendData`) crosses the line. |
| `meta` | Arbitrary payload returned by `getPriceLine` / `getPriceLines`. |

### Callback patterns

```js
// Draggable entry that reports its new price to your backend.
stock.addOrderLine({
  price: 98.5,
  side: 'buy',
  draggable: true,
  onMove: ({ id, price }) => saveOrder(id, price),
});

// Alert that fires once a streamed bar crosses it.
stock.addAlert({
  price: 100,
  closable: true,
  onCross: ({ direction, price }) => notify(`Crossed ${direction} at ${price}`),
  onRemove: ({ id }) => cancelAlert(id),
});
```

`onCross` fires from bars delivered via `appendData` (streaming), so wire alerts
before you start streaming.

## Annotations (data-space)

Id-based marks at **data coordinates** (price/time): lines, bands, point
markers, and text. Distinct from the drawing tools and the trading price
lines; they persist across update/theme/chart-type switches and are removed by
id, so they never disturb price lines or indicator annotations.

```js
stock.addAnnotation({ type: 'yLine', y: 128.5, label: 'Resistance' });
stock.addAnnotation({ type: 'yBand', y: 120, y2: 124, opacity: 0.15 });
stock.addAnnotation({ type: 'xLine', x: '2024-03-20', label: 'Earnings' });
stock.addAnnotation({ type: 'xBand', x: '2024-04-01', x2: '2024-04-08' });
const id = stock.addAnnotation({ type: 'point', x: t, y: 131, label: 'Buy' });
stock.addAnnotation({ type: 'text', x: t, y: 118, text: 'support zone' });

stock.updateAnnotation(id, { y: 130 });  // patch
stock.removeAnnotation(id);
stock.getAnnotation(id);                  // one config
stock.getAnnotations();                   // all configs
stock.clearAnnotations();                 // removes only annotations added this way
```

| Type | Required coords | Notes |
|---|---|---|
| `yLine` | `y` | Horizontal line. |
| `yBand` | `y`, `y2` | Horizontal shaded band. |
| `xLine` | `x` | Vertical line (`x` is a timestamp/date/category). |
| `xBand` | `x`, `x2` | Vertical shaded band. |
| `point` | `x`, `y` | Marker (customize via `marker`). |
| `text` | `x`, `y` | Text label, no marker. |

Common fields: `label`/`text`, `color`, `fillColor`, `opacity`, `textColor`,
`strokeDashArray`, `width`, `labelPosition`, `marker`, `meta`, and a stable
`id` (auto-generated when omitted).

## Drawings (price/time-anchored)

Trend lines, rays, levels, zones, Fibonacci levels, and measurements, anchored
in **data coordinates** so they re-project through zoom, pan, and resize.
Create them **with the mouse** (the on-chart toolbar has a tool for each
anchored type; the former "Line" tool is labeled "Trend line") or **from
code** with `addDrawing()`. Both share one model: mouse-drawn and programmatic
drawings drag, reproject, appear in `getDrawings()`, and serialize through
`getState()` / `setState()` identically.

```js
const id = stock.addDrawing({
  type: 'trendline',
  points: [{ x: '2024-02-01', y: 118 }, { x: '2024-04-15', y: 134 }],
  color: '#00b746',
  width: 2,
});

stock.addDrawing({ type: 'horizontalLine', points: [{ y: 128.5 }], dashArray: 4 });
stock.addDrawing({ type: 'verticalLine', points: [{ x: '2024-03-20' }] });
stock.addDrawing({
  type: 'rectangle',                       // supply/demand zone
  points: [{ x: '2024-03-01', y: 120 }, { x: '2024-03-20', y: 126 }],
  fill: '#008FFB', fillOpacity: 0.15,
});
stock.addDrawing({
  type: 'fibRetracement',                  // labeled level lines between two anchors
  points: [{ x: '2024-02-01', y: 112 }, { x: '2024-04-15', y: 138 }],
});
stock.addDrawing({
  type: 'measure',                         // box labeled with price change, % change, bar count
  points: [{ x: '2024-03-01', y: 118 }, { x: '2024-03-20', y: 129 }],
});
stock.addDrawing({
  type: 'trendline',
  points: [{ x: '2024-02-01', y: 0 }, { x: '2024-04-15', y: 0 }],
  snap: 'close',                           // snap each point to the nearest bar's close
});

stock.updateDrawing(id, { color: '#e91e63', visible: false });
stock.getDrawing(id);    // one drawing config
stock.getDrawings();     // all drawings, mouse-drawn shapes included
stock.removeDrawing(id);
stock.clearDrawings();
```

| Type (aliases) | Points | Notes |
|---|---|---|
| `trendline` (`line`) | 2 | Segment between two points. Draggable. |
| `ray` | 2 | Half-line from point 0 through point 1, extended to the edge. |
| `horizontalLine` (`hline`) | 1 | Price level spanning the full width (uses `y`). |
| `verticalLine` (`vline`) | 1 | Time marker spanning the full height (uses `x`). |
| `rectangle` (`zone`) | 2 | Filled box between two corner points. |
| `fibRetracement` | 2 | Fibonacci level lines between two anchor prices (labeled). |
| `fibExtension` | 2 | Fibonacci extension levels (ratios beyond 1). |
| `measure` | 2 | Box labeled with the price change, percent change, and bar count, tinted by direction. |

- `points` are `{ x, y }` in data space (`x` timestamp/date/category, `y`
  price).
- Common fields: `color`, `width`, `fill`, `fillOpacity`, `dashArray`,
  `locked` (not selectable/draggable), `visible` (still serialized when
  false), `meta`, and a stable `id` (auto-generated when omitted).
- `snap: true` or `snap: 'open' | 'high' | 'low' | 'close'` snaps each point
  to the nearest bar's OHLC values.
- The fib types accept `levels` (custom ratio array) and `showLabels`
  (default true); `measure` accepts `upColor` / `downColor` / `showLabel`.
- The line-family drawings (trend/ray/level/time/fib/measure) are draggable in
  the UI.
- **Two-anchor drawings can be reshaped (0.5.0).** Selecting a measure box,
  trend line, ray or fib puts a drag handle on **each of its two anchors**:
  drag one to change what it covers from that end. For a measurement, the
  statistics recompute over the bars it now spans. Previously a two-anchor
  drawing could only be translated, so re-measuring meant deleting and
  redrawing. Dragging the **body** still translates with the span intact, which
  is deliberate: that is how a measured move is projected onto a later
  breakout. A `locked: true` drawing shows no handles.
- Add/patch/remove/clear emit `drawingAdded` / `drawingUpdated` /
  `drawingRemoved` / `drawingsCleared` events (see SKILL.md §10).
- The whole drawing set is captured by `getState()` and restored by
  `setState()` (state schema v2). Measurements ride here too: a measurement
  *is* a `measure` drawing, so it needs no state key of its own. See
  [analysis.md](analysis.md) for `measureRange()` and the analysis panel.

Two 0.5.0 interaction fixes, in case you carry a workaround: **selecting a
drawing no longer makes it undraggable** (a blanket `pointer-events: all` on
overlay shapes used to beat the selection outline's `pointer-events="none"`, so
the outline swallowed every click meant for the element underneath), and **the
style popup no longer opens on top of the drawing it belongs to** (it is now
placed below, above, right or left, whichever fits inside the chart).

### Custom drawing tools: `ApexStock.registerDrawingTool(name, def)`

Register a new data-space drawing type globally (the drawing-layer analogue of
`registerIndicator`), then create it with `addDrawing({ type: name, points,
...customFields })`. It reprojects, drags, and serializes like a built-in.

```js
ApexStock.registerDrawingTool('markerDot', {
  defaults: { radius: 6 },
  render(data, helpers) {
    const p = helpers.dataToScreen(data.points[0].x, data.points[0].y);
    const c = document.createElementNS(helpers.svgNS, 'circle');
    c.setAttribute('cx', p.x);
    c.setAttribute('cy', p.y);
    c.setAttribute('r', data.radius);
    c.setAttribute('fill', data.color);
    return c;
  },
});

stock.addDrawing({ type: 'markerDot', points: [{ x: someTime, y: 132 }], color: '#e91e63' });
```

`render(data, helpers)` returns an SVG element from the drawing's data-space
record; `helpers` exposes `svgNS`, `dataToScreen(x, y)`, `screenToData(x, y)`,
`getChartBounds()`, and `extendToBounds(p1, p2)`. Custom types may not shadow
a built-in name; pass `overwrite: true` to replace a previous registration.
**A serialized custom drawing re-renders after reload only if the tool has
been registered again first** (call `registerDrawingTool` before `setState`).

## Comparison mode (multi-symbol)

Overlay additional instruments (e.g. AAPL vs MSFT vs SPY) as lines on a
dedicated **secondary y-axis**, so differing price scales never distort the
primary candlestick. Do **not** add extra symbols as extra `series` entries
(those are treated as indicators).

```js
stock.addComparison({ name: 'MSFT', data: msftBars }); // data: [{x, y}] closes or full OHLC (close is used)
stock.addComparison({ name: 'SPY', data: spyBars, color: '#FEB019' });

stock.setComparisonMode('percent');   // default
stock.setComparisonMode('absolute');  // raw prices instead
stock.getComparisonMode();

stock.getComparisons();               // -> [{ name, color, points }]
stock.removeComparison('SPY');
stock.clearComparisons();
```

- **`percent`** (default): percent change from the baseline, so you compare
  performance ("who's up more") regardless of nominal price.
- **`absolute`**: raw close prices on the secondary axis (best for same-scale
  peers).
- You supply the compared instrument's data (ApexStock does not fetch it).
- Comparisons persist across zoom, theme change, chart-type switch, indicator
  toggles, and `appendData`.
- **A comparison instrument may not take the primary series' name (0.5.0)**, nor
  the reserved `'__primary__'`. Previously that filtered the price series off
  the chart.

> **Comparison was rebuilt in 0.5.0** and is much larger than the four calls
> above: instruments are aligned onto one shared x grid before anything is
> normalized (`join` / `fill`), 0% can sit in four different places
> (`baseline`, whose **default changed** from each instrument's own first point
> to `'common'`), there are three new modes (`indexed`, `relative`, `ratio`),
> the benchmark is a role you can point at any instrument, and
> `getComparisonStats()` returns a render-ready leaderboard. All of it is in
> **[analysis.md](analysis.md)**. The `percent` / `absolute` pair above still
> works exactly as before.

## Event markers (0.5.0)

Time-anchored flags that float along the x-axis with a hover card. Drawn on a
lightweight HTML overlay that reprojects through zoom and pan and hides markers
scrolled off the visible range.

```js
const id = stock.addEventMarker({
  x: '2024-02-01',       // epoch ms | ISO string | Date
  type: 'earnings',      // 'earnings' | 'dividend' | 'split' | 'news' | 'custom'
  label: 'Q4 beat',
  color: undefined,      // defaults to the type's own colour
  glyph: undefined,      // defaults to the type's own glyph
  position: 'top',       // 'top' | 'bottom'
  meta: { eps: 2.18 },   // anything; handed back on hover/click
});

stock.updateEventMarker(id, { label: 'Q4 beat (revised)' });
stock.removeEventMarker(id);
stock.clearEventMarkers();
stock.getEventMarker(id);
stock.getEventMarkers();
```

Each of the five types has a default glyph and colour. Markers are captured by
`getState()` and restored by `setState()` (state schema v2 gains an
`eventMarkers` field).

**Events:** `eventMarkerAdded` / `eventMarkerUpdated` (`{ id, marker }`),
`eventMarkerRemoved` (`{ id }`), `eventMarkersCleared` (`{}`), and
`eventMarkerHover` / `eventMarkerClick` (`{ id, marker, nativeEvent }`).

## On-chart data legend (0.5.0)

A corner panel that reads out the instrument's OHLC, change (vs the previous
close), and volume at the crosshair, falling back to the latest bar, plus the
value of each main-chart **overlay** indicator.

```js
new ApexStock(el, {
  legend: {
    show: true,
    position: 'top-left',   // 'top-left' | 'top-right' | 'bottom-left' | 'bottom-right'
    showChange: true,
    showVolume: true,
    showIndicators: true,
  },
});

stock.showLegend({ position: 'bottom-right' });
stock.hideLegend();
stock.toggleLegend();      // -> the new visibility
stock.isLegendVisible();
```

It tracks the pointer via the `crosshairMove` event, stays in sync as
indicators toggle, and is `pointer-events: none` so it never intercepts chart
interaction. A bottom corner clears the custom x-axis strip, so it means the
bottom of the **plot** (fixed in 0.5.0: the strip is opaque and painted above
the legend, so a `bottom-left` panel used to have its lower rows silently
sliced off).

For the same numbers programmatically, without the panel, use
`getDataAt(index)` (see [analysis.md](analysis.md)).

## Toolbar customization (0.5.0)

Hide the whole primary toolbar, hide individual built-in sections, or inject
your own controls.

```js
new ApexStock(el, {
  toolbar: {
    show: true,
    items: { chartType: true, indicators: true, drawing: true, download: false },
  },
});
```

Each built-in section is shown by default. Custom items:

```js
const id = stock.addToolbarItem({
  id: 'snapshot',
  title: 'Snapshot',
  icon: '📷',                       // or html: '<svg…>', or element: someNode
  position: 'right',                // 'left' | 'left-start' | 'right'
  order: 10,                        // sort within the position
  onClick: (chart, event) => chart.export({ format: 'png', download: true }),
});

stock.removeToolbarItem(id);
stock.getToolbarItems();   // -> [{ id, title, position }]
```

Custom items carry `.apexstock-toolbar-button`, so they follow the active theme
and any preset.

## Teardown

As of 0.4.0, `stock.destroy()` fully removes the toolbars, drawing overlay,
listeners, and panes (it is idempotent and StrictMode-safe). Only on ≤ 0.3.x
did the indicator toolbar and drawing-tools DOM survive `destroy()`; if you
must support those versions, wrap each chart in a dedicated element you own
and remove the wrapper on teardown.
