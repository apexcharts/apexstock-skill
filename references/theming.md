# Theming

## Light / dark mode

Set the palette at construction or switch it at runtime:

```js
const stock = new ApexStock(el, { series, theme: { mode: 'dark' } }); // or 'light'
stock.render();

stock.updateTheme('light');   // switch at runtime
stock.getTheme();             // -> 'light' | 'dark'
```

(Passing an explicit `theme: undefined` is stripped so ApexCharts applies its
default; omit the key rather than passing `undefined`.)

As of 0.5.0 `theme` reads two fields: `mode` and `preset`.

## Theme presets (0.5.0)

A curated, **light-first** pack of named themes layered on the base light/dark
modes: `paper`, `arctic`, `mint`, `linen`, `rose`, `graphite`.

```js
const stock = new ApexStock(el, { series, theme: { preset: 'mint' } });

stock.setThemePreset('graphite');
stock.getThemePreset();          // -> 'graphite' | null
ApexStock.getThemePresets();     // -> every known preset name
```

Each preset retints **both** surfaces at once, which is the point of the
feature:

- **the chart**: candles, grid, axis labels, plot background;
- **the chrome**: toolbar, dropdowns, legend, via the `--apx-*` family tokens it
  sets on the container.

Trading price lines follow the accent. The indicator palette stays the base
mode's, so an RSI does not change colour when you switch presets.

**Lifecycle.** A preset carries its own base mode, survives theme and
chart-type switches and data updates, and is captured by `getState()` as
`theme.preset`. It is **cleared by `updateTheme()`**: switching the raw mode is
how you drop back to an unthemed light or dark.

### Registering your own

```js
ApexStock.registerTheme('brand', {
  mode: 'dark',        // the base mode this preset sits on
  up: '#2ec27e',
  down: '#e01b24',
  grid: '#2a2f3a',
  axis: '#8b93a1',
  background: '#12161d',
  accent: '#7c3aed',
});

stock.setThemePreset('brand');
```

Any subset of `{ mode, up, down, grid, axis, background, accent }` works; the
rest is backfilled from the mode. Custom presets take precedence over built-ins
of the same name.

## Restyling with CSS custom properties

To go beyond light/dark, change the accent color, radii, fonts, borders, override
the `--apexstock-*` CSS custom properties. **No stylesheet fork required.**

The tokens are declared on `[class^="apexstock-"]` (the toolbar/UI wrapper
classes), **not** on `:root`, so they are self-contained: they never leak into
your page and your page's own design tokens never leak into the chart. This is
why an ApexStock chart is safe to drop into any design system.

```css
/* Scope overrides to your chart's container (recommended). */
#my-chart [class^='apexstock-'] {
  --apexstock-blue: #7c3aed;         /* accent */
  --apexstock-font-size-sm: 12px;
  --apexstock-light-bg: #ffffff;
  --apexstock-dark-bg: #0b0f14;
}
```

Token families include background/surface, border, hover, text, muted, selected,
divider, and shadow variants for both `--apexstock-light-*` and
`--apexstock-dark-*`, plus accent (`--apexstock-blue`) and sizing tokens.

A copy-ready starting point ships with the core library:

```js
import 'apexstock/theme-template.css';   // then edit the token values
```

See the library's `THEMING.md` and `examples/theming.html` for the full token
reference and a live override recipe.

## Theming bugs fixed in 0.5.0 (do not work around these)

Four surfaces used to be painted **inline at construction time**, which meant
they could not follow a later `updateTheme()` or `setThemePreset()`. If your
app carries a workaround for any of these, remove it.

- **The chart container kept its construction-time surface.** It was painted
  inline from `colors.toolbar.background`, so a chart switched to dark kept a
  white container around it. Worse, the same inline colour was written onto the
  container's **parent**, an element you own, overriding your stylesheet and
  then going stale. The surface now comes from the stylesheet, keyed off a new
  `apexstock-chart` marker class on the container, so it follows both the mode
  and a preset's `--apx-surface`.
- **The custom x-axis stayed in the light palette after `updateTheme('dark')`.**
  Its surface and top rule were assigned inline at construction; tick colours
  had the mirror-image problem, read per render, so they only caught up on the
  next zoom or pan. Both now come from the stylesheet, which is also what lets
  a preset retint the axis.
- **The price crosshair label did not match the date chip.** ApexCharts draws
  the price (y) readout and ApexStock draws the date (x) one. The price label is
  now restyled by setting ApexCharts' four `--apx-axt-*` tokens, which its
  pointer triangle derives its colours from too, so the whole label retints with
  no geometry to override. Both readouts resolve the same `--apexstock-*`
  tokens, so they follow a mode change and a preset together.
- **The x-axis crosshair readout has been redrawn.** It was a pale bordered box
  pinned to one palette; it is now a filled cell spanning the axis strip (the
  fill is the text colour and the text is the surface colour, so it inverts with
  the theme and follows a preset's `--apx-fore` / `--apx-surface`), set in
  tabular figures so the chip does not resize under the cursor. Axis tick labels
  now read muted rather than at full text strength.

## Historical note

Before 0.2.0 these tokens were un-namespaced (`--font-size-sm`, `--blue`, …) and
declared on the global `:root`, which clobbered host design tokens of the same
name (most visibly shrinking host text via `--font-size-sm: 10px`). As of 0.2.0
every token is prefixed `--apexstock-*` and scoped, so this no longer happens.
Do not write CSS that works around the old behavior.
