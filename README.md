# ApexStock AI Skill

AI coding skill for building [ApexStock](https://apexcharts.com/docs/apexstock/) financial / stock charts. Works with Claude Code, Cursor, GitHub Copilot, and any AI coding assistant that can read project context.

> **Separate skill, one of the ApexCharts ecosystem skills.** This is the dedicated skill for **ApexStock** (`apexstock`), shipped as its own `apexstock-skill` package and repo, distinct from the core `apexcharts-skill` and the other product skills. Each product has its own library and skill; use the one that matches yours:
>
> | Product | npm library | Skill package & repo |
> |---|---|---|
> | ApexCharts, charts | `apexcharts` | [`apexcharts-skill`](https://github.com/apexcharts/apexcharts-skill) |
> | ApexGantt, Gantt / timeline | `apexgantt` | [`apexgantt-skill`](https://github.com/apexcharts/apexgantt-skill) |
> | ApexTree, hierarchy / org charts | `apextree` | [`apextree-skill`](https://github.com/apexcharts/apextree-skill) |
> | ApexSankey, flow / Sankey | `apexsankey` | [`apexsankey-skill`](https://github.com/apexcharts/apexsankey-skill) |
> | Apex Grid, data grid | `apex-grid` | [`apexgrid-skill`](https://github.com/apexcharts/apexgrid-skill) |
| ApexMaps, maps / geo | `apexmaps` | [`apexmaps-skill`](https://github.com/apexcharts/apexmaps-skill) |
> | **ApexStock**, financial / stock · *this skill* | `apexstock` | `apexstock-skill` |

## What This Does

AI models routinely get stock-chart code wrong: forgetting to provide the ApexCharts constructor, using flat `o`/`h`/`l`/`c` keys instead of the `y: [o,h,l,c]` array, calling `update()` on every tick instead of `appendData()`, or adding a second symbol as another `series` entry instead of comparison mode. This skill ships structured reference files so the assistant generates correct ApexStock code on the first try.

> **ApexStock 0.5.0 requires `apexcharts` `^7.1.0`** (it was `^6.7.0` for 0.4.0). This is a breaking change, and the wrappers at 0.2.2 carry it too. The skill leads with it.

### Coverage

- **OHLC data format**, `{ x, y: [open, high, low, close], v? }`, chart types, timestamps, normalization, the data adapters (`normalize` / `fromArrays` / `fromCSV`), and `exportData` / `exportImage`
- **Providing ApexCharts**, per-instance injection (`{ ApexCharts }`), app-wide `setApexCharts`, or the `window.ApexCharts` global
- **Technical indicators**, overlays vs oscillators (multiple oscillator panes since 0.4.0), the full key list incl. VWAP / ATR / Donchian / Keltner, `updateIndicator(key, params)` / `setIndicatorParams`, introspection, custom indicators, `calculate*` helpers
- **Real-time streaming**, `appendData`, `view` / `maxPoints` / `updateLast`, forming-candle recipes, visible-range accessors, `aggregateOHLC`
- **Trading overlays**, order / stop-loss / take-profit / alert price lines with drag / close / cross callbacks
- **Drawings & annotations**, the price/time-anchored `addDrawing` API (trend lines, rays, Fibonacci, measure, `snap`, `registerDrawingTool`) and data-space annotations (`yLine` / `yBand` / `xLine` / `xBand` / `point` / `text`)
- **Comparison mode**, multi-instrument overlays with alignment policies, four baseline policies, five modes (absolute / percent / indexed / relative / ratio), the benchmark role, and the `getComparisonStats()` leaderboard
- **Analysis workspace** (0.5.0), `getRangeStats` / `getDrawdown` / the drawdown pane / per-pane heights / `measureRange` and the analysis panel / `getDataAt` / price-scale modes / the headless `ApexStock.stats` namespace
- **Chart furniture** (0.5.0), event markers, the on-chart data legend, toolbar customization, and `ApexStock.sync()` for linking independent charts
- **Events & state**, `on` / `off` / `once`, `crosshairMove` / `rangeChange` / `rangeChanging` / `indicatorToggle` / `drawing*` / `eventMarker*` / `priceScaleChange` / `rangeMeasured` / `comparison*`, and `getState` / `setState` (schema v2 now persists annotations, price lines, event markers, price scale, pane ratios, and the comparison setup)
- **Export**, unified `export()` over png / svg / **pdf** / csv / json, with `include: ['indicators', 'analysis']`
- **Theming**, light/dark modes, the named preset pack + `registerTheme`, and the scoped `--apexstock-*` token system
- **Framework wrappers**: `react-apexstock`, `vue-apexstock`, `ngx-apexstock`, incl. the `apexCharts` injection prop

## Installation

### Claude Code

```bash
mkdir -p .claude/skills
cd .claude/skills
git clone https://github.com/apexcharts/apexstock-skill.git
```

### Cursor / Windsurf

```bash
curl -o .cursorrules https://raw.githubusercontent.com/apexcharts/apexstock-skill/main/.cursorrules
```

### GitHub Copilot

Reference `SKILL.md` in Copilot Chat: `@workspace #file:SKILL.md`, or paste the contents of `.cursorrules` into Copilot's custom instructions.

### Generic AI Assistant

Paste the contents of `SKILL.md` into the system prompt or attach it as context.

### As an npm dependency

For tools that build on top of this skill (MCP servers, custom AI agents):

```bash
npm install apexstock-skill
```

```js
import { skillFile, referencesDir, referencePath } from 'apexstock-skill';
import { readFile } from 'node:fs/promises';

const skill = await readFile(skillFile, 'utf8');
const streaming = await readFile(referencePath('streaming-and-aggregation.md'), 'utf8');
```

## Repository Structure

```
├── SKILL.md                            # Main entry point, read this first
├── .cursorrules                        # Self-contained version for Cursor / Windsurf
├── references/
│   ├── analysis.md                     # statistics, drawdown, measurements, comparison v2, price scale
│   ├── data-format.md                  # OHLC points, chart types, adapters, export
│   ├── indicators.md                   # overlays vs oscillators, keys, params, panes, custom indicators
│   ├── state-and-export.md             # getState/setState v2, export() incl. PDF, ApexStock.sync
│   ├── streaming-and-aggregation.md    # appendData, forming candle, visible range, aggregateOHLC
│   ├── trading-overlays.md             # price lines, annotations, drawings, event markers, legend, toolbar
│   ├── framework-wrappers.md           # React, Vue, Angular
│   └── theming.md                      # light/dark, theme presets, --apexstock-* tokens
└── install/
    ├── claude-code.md
    ├── cursor.md
    └── copilot.md
```

## Links

- [ApexStock Documentation](https://apexcharts.com/docs/apexstock/)
- [ApexStock GitHub](https://github.com/apexcharts/apexstock)
- [npm: apexstock](https://www.npmjs.com/package/apexstock)
- [react-apexstock](https://www.npmjs.com/package/react-apexstock)
- [vue-apexstock](https://www.npmjs.com/package/vue-apexstock)
- [ngx-apexstock](https://www.npmjs.com/package/ngx-apexstock)

## License

MIT
