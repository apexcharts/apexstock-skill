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
> | **ApexStock**, financial / stock · *this skill* | `apexstock` | `apexstock-skill` |

## What This Does

AI models routinely get stock-chart code wrong: forgetting that ApexCharts must be a global, using flat `o`/`h`/`l`/`c` keys instead of the `y: [o,h,l,c]` array, calling `update()` on every tick instead of `appendData()`, or expecting several oscillators at once. This skill ships structured reference files so the assistant generates correct ApexStock code on the first try.

### Coverage

- **OHLC data format**, `{ x, y: [open, high, low, close], v? }`, chart types, timestamps, normalization
- **The ApexCharts-global requirement**, why `window.ApexCharts` must be set first
- **Technical indicators**, overlays vs oscillators, the full key list, `period` / `stdDev` config, `calculate*` helpers
- **Real-time streaming**, `appendData`, `view` / `maxPoints` / `updateLast`, forming-candle recipes, `aggregateOHLC`
- **Trading overlays**, order / stop-loss / take-profit / alert price lines with drag / close / cross callbacks
- **Theming**, light/dark modes and the scoped `--apexstock-*` token system
- **Lifecycle & the 0.3.0 `destroy()` teardown gap**, with the wrapper-element remediation
- **Framework wrappers**: `react-apexstock`, `vue-apexstock`, `ngx-apexstock`

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
│   ├── data-format.md                  # OHLC points, chart types, timestamps
│   ├── indicators.md                   # overlays vs oscillators, keys, calculate* helpers
│   ├── streaming-and-aggregation.md    # appendData, forming candle, aggregateOHLC
│   ├── trading-overlays.md             # price lines, drag/close/cross, drawing tools
│   ├── framework-wrappers.md           # React, Vue, Angular
│   └── theming.md                      # light/dark, --apexstock-* tokens
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
