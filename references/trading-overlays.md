# Trading Overlays (price lines) & Drawing Tools

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

## Drawing tools

The toolbar includes interactive drawing tools (trend lines, Fibonacci, text
annotations, etc.) for manual technical analysis. They are mounted by `render()`
automatically; no config is required to enable the toolbar.

> **Teardown note (0.3.0):** the drawing-tools toolbar/overlay and the indicator
> toolbar are **not** removed by `stock.destroy()`. In a single-page app that
> mounts/unmounts charts, wrap each chart in a dedicated element you own and
> remove that wrapper on teardown so the leftover toolbars go with it. See
> SKILL.md §9 for the pattern.
