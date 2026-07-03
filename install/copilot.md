# Installing ApexStock Skill for GitHub Copilot

## VS Code + Copilot Chat

### Option 1: Add as Workspace Context

1. Clone this repository or download `SKILL.md` into your project
2. In Copilot Chat, reference the file: `@workspace #file:SKILL.md`
3. Ask your stock-chart question

### Option 2: Custom Instructions

1. Open VS Code Settings (Cmd/Ctrl + ,)
2. Search for "Copilot Custom Instructions"
3. Paste the contents of `.cursorrules` into the custom instructions field

## Copilot in GitHub.com

When working in GitHub's web editor with Copilot, paste the relevant sections from `SKILL.md` into your prompt for accurate ApexStock code generation.

## Verification

Ask Copilot to generate a candlestick chart with Bollinger Bands and a live tick feed. It should set `window.ApexCharts` before constructing, use the `{ x, y: [o,h,l,c], v? }` OHLC shape, add indicators through `plotOptions.stockChart.indicators`, and stream bars with `appendData(...)` rather than `update()`.
