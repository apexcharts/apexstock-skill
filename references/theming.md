# Theming

## Light / dark mode

Set the palette at construction or switch it at runtime:

```js
const stock = new ApexStock(el, { series, theme: { mode: 'dark' } }); // or 'light'
stock.render();

stock.updateTheme('light');   // switch at runtime
stock.getTheme();             // -> 'light' | 'dark'
```

`theme.mode` is the only theme field ApexStock reads. (Passing an explicit
`theme: undefined` is stripped so ApexCharts applies its default, omit the key
rather than passing `undefined`.)

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

## Historical note

Before 0.2.0 these tokens were un-namespaced (`--font-size-sm`, `--blue`, …) and
declared on the global `:root`, which clobbered host design tokens of the same
name (most visibly shrinking host text via `--font-size-sm: 10px`). As of 0.2.0
every token is prefixed `--apexstock-*` and scoped, so this no longer happens.
Do not write CSS that works around the old behavior.
