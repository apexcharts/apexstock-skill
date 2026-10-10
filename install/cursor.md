# Installing ApexStock Skill for Cursor

## Setup

1. Copy the `.cursorrules` file from this repository into the root of your project:

```bash
cp /path/to/apexstock-skill/.cursorrules /path/to/your-project/.cursorrules
```

Or download it directly:

```bash
curl -o .cursorrules https://raw.githubusercontent.com/apexcharts/apexstock-skill/main/.cursorrules
```

2. Restart Cursor or open a new Cursor window.

Cursor automatically reads `.cursorrules` files in the project root and uses them as context for AI-assisted coding.

## For Windsurf

Same approach, Windsurf also supports `.cursorrules` files in the project root.

## Verification

Ask Cursor to generate a candlestick chart with indicators and streaming. It should provide the ApexCharts constructor (injected with `new ApexStock(el, options, { ApexCharts })`, registered with `ApexStock.setApexCharts`, or the `window.ApexCharts` global), use the `{ x, y: [o,h,l,c], v? }` data shape, follow the `render()` / `appendData()` / `destroy()` lifecycle, and know that several oscillators can be active at once, each in its own pane (since 0.4.0).
