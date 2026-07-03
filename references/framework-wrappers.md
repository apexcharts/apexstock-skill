# Framework Wrappers: React, Vue, Angular

Thin, typed component wrappers around the imperative `ApexStock` class. They
create the instance on mount, call `render()`, forward `options` / `series`
changes to `update()`, and call `destroy()` on unmount. Prefer them over the
core API in framework projects.

```bash
npm install apexstock react-apexstock   # React
npm install apexstock vue-apexstock     # Vue 3
npm install apexstock ngx-apexstock     # Angular
```

All three peer-depend on `apexstock` (`>=0.3.0 <1`) and on `apexcharts`.

## The ApexCharts-global rule still applies

The wrappers do **not** set `window.ApexCharts` for you. Set it once at app
startup, before any chart mounts:

```js
import ApexCharts from 'apexcharts';
if (typeof window !== 'undefined') window.ApexCharts = ApexCharts;
```

## Common shape

- **Props / inputs:** `options` (required, the full `StockChartOptions`) and
  `series` (optional; overrides `options.series`, handy for keeping static
  config in `options` and streaming data through `series`).
- **Accessors:** `getInstance()` → the live `ApexStock` (or `null`), and
  `getElement()` → the container `<div>`.
- **Updates diff by identity.** Pass a **new** object/array to trigger an
  `update()`; mutating the same object in place will not (avoids deep-watching
  large datasets). Memoize in React; produce new refs in Vue/Angular.
- **SSR-safe:** the instance is created client-side only; the server renders
  just the container `<div>`.

## React (`react-apexstock`)

```tsx
import { useMemo, useRef } from 'react';
import ApexStock, { type ApexStockRef } from 'react-apexstock';

function PriceChart({ candles }: { candles: OHLCPoint[] }) {
  const ref = useRef<ApexStockRef>(null);
  const options = useMemo(
    () => ({
      chart: { height: 600 },
      series: [{ name: 'AAPL', data: candles }],
      theme: { mode: 'light' as const },
      plotOptions: { stockChart: { indicators: { 'moving average': { enabled: true } } } },
    }),
    [candles],
  );

  // Imperative access when you need it:
  // ref.current?.getInstance()?.updateIndicator('rsi');

  return <ApexStock ref={ref} options={options} style={{ width: '100%' }} />;
}
```

Props: `options` (required), `series?`, `className?`, `style?`. The ref exposes
`getInstance()` and `getElement()`.

## Vue 3 (`vue-apexstock`)

```vue
<script setup lang="ts">
import { ref, shallowRef } from 'vue';
import ApexStock from 'vue-apexstock';

const chart = ref();
const options = shallowRef({
  chart: { height: 600 },
  series: [{ name: 'AAPL', data: candles }],
  theme: { mode: 'light' },
});

// chart.value.getInstance()?.appendData(bar);
</script>

<template>
  <ApexStock ref="chart" :options="options" />
</template>
```

Props: `options` (required), `series?`. Exposes `getInstance()` /
`getElement()`. Reassign `options` (or pass a new `series`) to update, an
in-place mutation won't trigger a redraw.

## Angular (`ngx-apexstock`)

Standalone component with selector `apex-stock`, import it directly into a
component's `imports` (no module needed).

```ts
import { Component, ViewChild } from '@angular/core';
import { ApexStockComponent } from 'ngx-apexstock';

@Component({
  selector: 'app-price-chart',
  standalone: true,
  imports: [ApexStockComponent],
  template: `<apex-stock [options]="options" [series]="series"></apex-stock>`,
})
export class PriceChartComponent {
  @ViewChild(ApexStockComponent) chart!: ApexStockComponent;

  options = { chart: { height: 600 }, theme: { mode: 'light' as const } };
  series = [{ name: 'AAPL', data: candles }];

  // this.chart.getInstance()?.updateTheme('dark');
}
```

Inputs: `options` (required), `series?`. Methods: `getInstance()`,
`getElement()`.

## Cleanup caveat (0.3.0)

The wrappers call `stock.destroy()` on unmount, but the core's `destroy()` does
not remove the indicator toolbar or the drawing-tools DOM (they are inserted
outside the component's own container). On a route that mounts/unmounts the chart
repeatedly, these can accumulate. Until it's fixed upstream, mount the chart in a
view/route whose whole subtree is torn down, or clear the container's parent on
teardown. See SKILL.md §9.
