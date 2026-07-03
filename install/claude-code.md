# Installing ApexStock Skill for Claude Code

## Installation

```bash
# Navigate to your project's Claude config
mkdir -p .claude/skills
cd .claude/skills

# Clone the skill
git clone https://github.com/apexcharts/apexstock-skill.git
```

Claude Code will automatically detect `SKILL.md` and load it when working on ApexStock code.

## Verification

Ask Claude to build a stock chart:

> Create an ApexStock candlestick chart with a 20-period moving average and an RSI oscillator, then stream a new bar into it.

Claude should generate code that:
- Sets `window.ApexCharts = ApexCharts` before constructing (ApexStock needs the global)
- Calls `new ApexStock(el, options)` followed by `stock.render()`
- Uses the OHLC data format `{ x, y: [open, high, low, close], v? }` (single `y` array, not `o`/`h`/`l`/`c`)
- Enables indicators via `plotOptions.stockChart.indicators` (or `updateIndicator('rsi')`)
- Uses `appendData(...)`, not `update()`, for the streamed bar
