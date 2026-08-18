# Real-time Streaming & Timeframe Aggregation

## `appendData`: the streaming fast path

```js
stock.appendData(pointOrPoints, options);
```

`appendData` adds a bar (or a batch), or replaces the forming bar, and refreshes
price candles, every active overlay and oscillator pane, the volume pane, and
the x-axis **incrementally**, `O(active indicators × small tail)` instead of
the full teardown/rebuild `update()` performs. It returns the instance.

- **`pointOrPoints`**, one OHLC bar `{ x, y: [o,h,l,c], v? }`, or an array for
  a batch. Malformed points are dropped, like the constructor.
- **`options.view`**, `"follow"` (default) keeps the newest bars in view
  (shifts a zoomed window toward the right edge); `"preserve"` keeps the current
  zoom window fixed.
- **`options.maxPoints`**, rolling-window cap: trims the oldest bars off the
  front so the buffer stays a fixed width. Running indicators keep their carried
  state, so their values reflect all history seen (they don't jump when old bars
  age out, intentionally different from a cold reload of the truncated buffer).
- **`options.updateLast`**, when the incoming `x` equals the last bar's `x`,
  **replace** that (forming) bar instead of appending. A new-`x` bar is then
  treated as still forming.

```js
// A completed bar arrived → append and ride the right edge.
stock.appendData({ x: t, y: [o, h, l, c], v }, { view: 'follow' });

// Live ticker with a fixed 500-bar window.
stock.appendData(bar, { maxPoints: 500 });
```

**Use `appendData`, not `update()`, for live data.** `update()` is for full
reconfigures (new dataset, theme, options).

Every built-in indicator has an exact **streaming twin** that steps forward
from committed state on each append, including the 0.4.0 additions: VWAP stays
exact in O(1), and ATR / Donchian Channels / Keltner Channels step
incrementally (Keltner composes the EMA and ATR steppers). Custom indicators
registered without a `stream` twin are recomputed from the full series on each
append (still exact, but O(n); see
[indicators.md](indicators.md)).

## Tick-to-bar recipes

ApexStock renders bars; you build them from a raw tick/trade feed. Two patterns:

### 1. Append completed bars

Roll trades into bars yourself (or with `ApexStock.aggregateOHLC`), then append
each finished bar:

```js
// Roll a 1-minute series up to 5-minute bars, then stream the closed ones.
const fiveMin = ApexStock.aggregateOHLC(oneMinSeries, '5m');
stock.appendData(fiveMin[fiveMin.length - 1]);
```

### 2. Forming candle (`updateLast`)

Keep the in-progress bar live as ticks arrive; start a new bar when the period
rolls over:

```js
let bar = null; // the bar currently forming

function onTrade({ time, price, size }) {
  const bucket = Math.floor(time / 60000) * 60000; // 1-minute buckets
  if (!bar || bar.x !== bucket) {
    // New period: the previous bar has closed; start a fresh forming bar.
    bar = { x: bucket, y: [price, price, price, price], v: size };
  } else {
    // Same period: fold the trade into the forming bar.
    bar.y[1] = Math.max(bar.y[1], price); // high
    bar.y[2] = Math.min(bar.y[2], price); // low
    bar.y[3] = price;                      // close
    bar.v += size;
  }
  stock.appendData(bar, { updateLast: true });
}
```

Both keep indicators exact: a forming bar's indicator values recompute from the
last committed state each tick, and the bar's close commits in O(1).

## Visible range & lazy loading

Read or set the visible x-window programmatically (0.4.0):

```js
const range = stock.getVisibleRange();          // { min, max } or null
stock.setVisibleRange(Date.UTC(2024, 0, 1), Date.UTC(2024, 5, 30));
```

`setVisibleRange(min, max)` zooms the main chart **and every indicator pane**
and fires the `rangeChange` event, exactly like an interactive zoom.
`getVisibleRange()` reports the same values the `rangeChange` event carries.
Typical lazy-loading loop: subscribe to `rangeChange`, and when the user pans
near the left edge, fetch older bars and `update({ series })` (or prepend and
`setVisibleRange` back to where they were).

```js
stock.on('rangeChange', ({ min, max, source }) => {
  if (min <= earliestLoaded) loadOlderBars(min);
});
```

## Timeframe aggregation

`ApexStock.aggregateOHLC` is a **pure** static helper, pass the result to
`new ApexStock(...)` or `update({ series })`:

```js
const hourly = ApexStock.aggregateOHLC(oneMinuteSeries, '1h');
const daily  = ApexStock.aggregateOHLC(hourly, '1d');
```

Accepted intervals (`ApexStock.INTERVALS`):

```
1m  5m  15m  30m  1h  2h  4h  12h  1d  1w  1M
```

Fixed-width intervals bucket on **UTC** epoch boundaries (`1d` = UTC midnight,
`4h` = 00/04/08/… UTC). `1w` is UTC-week Monday-anchored; `1M` is the calendar
month. A typical timeframe switcher aggregates a base 1-minute series to the
selected interval and calls `update({ series: [{ data: aggregated }] })`.
