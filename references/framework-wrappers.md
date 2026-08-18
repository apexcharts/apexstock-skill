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

All three peer-depend on `apexstock` (`>=0.3.0 <1`) and on `apexcharts`
(`^6.7.0` as of ApexStock 0.4.0).

## Providing ApexCharts: use the `apexCharts` prop

ApexStock needs the ApexCharts constructor but does not bundle it. As of
0.4.0, every wrapper takes an optional **`apexCharts` prop/input**, pass it
the imported constructor and skip the `window` global entirely:

```js
import ApexCharts from 'apexcharts';
// <ApexStock options={options} apexCharts={ApexCharts} />        (React)
// <ApexStock :options="options" :apex-charts="ApexCharts" />     (Vue)
// <apex-stock [options]="options" [apexCharts]="ApexCharts" />   (Angular)
```

Alternatives (resolution order per instance): the injected constructor, then
an app-wide `ApexStock.setApexCharts(ApexCharts)` registered once at startup,
then the `window.ApexCharts` global. Setting the global still works and is the
only option on ApexStock ≤ 0.3.x.

## Common shape

- **Props / inputs:** `options` (required, the full `StockChartOptions`),
  `series` (optional; overrides `options.series`, handy for keeping static
  config in `options` and streaming data through `series`), and `apexCharts`
  (optional; the ApexCharts constructor, forwarded at mount).
- **Accessors:** `getInstance()` → the live `ApexStock` (or `null`), and
  `getElement()` → the container `<div>`.
- **Updates diff by identity.** Pass a **new** object/array to trigger an
  `update()`; mutating the same object in place will not (avoids deep-watching
  large datasets). Memoize in React; produce new refs in Vue/Angular.
- **SSR-safe:** the instance is created client-side only; the server renders
  just the container `<div>`.
- **Clean unmount:** the wrappers call `destroy()` on unmount; as of core
  0.4.0 that is a complete, idempotent teardown (safe under React
  StrictMode's double-invoke).

## React (`react-apexstock`)

```tsx
import { useMemo, useRef } from 'react';
import ApexCharts from 'apexcharts';
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

  return <ApexStock ref={ref} options={options} apexCharts={ApexCharts} style={{ width: '100%' }} />;
}
```

Props: `options` (required), `series?`, `apexCharts?`, `className?`, `style?`.
The ref exposes `getInstance()` and `getElement()`.

## Vue 3 (`vue-apexstock`)

```vue
<script setup lang="ts">
import { ref, shallowRef } from 'vue';
import ApexCharts from 'apexcharts';
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
  <ApexStock ref="chart" :options="options" :apex-charts="ApexCharts" />
</template>
```

Props: `options` (required), `series?`, `apexCharts?`. Exposes `getInstance()`
/ `getElement()`. Reassign `options` (or pass a new `series`) to update, an
in-place mutation won't trigger a redraw.

## Angular (`ngx-apexstock`)

Standalone component with selector `apex-stock`, import it directly into a
component's `imports` (no module needed).

```ts
import { Component, ViewChild } from '@angular/core';
import ApexCharts from 'apexcharts';
import { ApexStockComponent } from 'ngx-apexstock';

@Component({
  selector: 'app-price-chart',
  standalone: true,
  imports: [ApexStockComponent],
  template: `<apex-stock [options]="options" [series]="series" [apexCharts]="apexCharts"></apex-stock>`,
})
export class PriceChartComponent {
  @ViewChild(ApexStockComponent) chart!: ApexStockComponent;

  apexCharts = ApexCharts;
  options = { chart: { height: 600 }, theme: { mode: 'light' as const } };
  series = [{ name: 'AAPL', data: candles }];

  // this.chart.getInstance()?.updateTheme('dark');
}
```

Inputs: `options` (required), `series?`, `apexCharts?`. Methods:
`getInstance()`, `getElement()`.

## Cleanup on old cores (≤ 0.3.x only)

With core ApexStock 0.4.0+, unmounting is fully clean: `destroy()` removes the
underlying charts, panes, toolbars, listeners, and observers, and is safe to
call more than once. If the project pins an older core (≤ 0.3.x), `destroy()`
there did not remove the indicator toolbar or the drawing-tools DOM; on such
versions mount the chart in a view/route whose whole subtree is torn down, or
clear the container's parent on teardown.
