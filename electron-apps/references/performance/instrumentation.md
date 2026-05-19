# Performance Instrumentation & Profiling

## Chrome DevTools Profiling

Electron uses Chromium — full Chrome DevTools available. Open via `View → Toggle Developer Tools` or `win.webContents.openDevTools()`.

**Key panels:**
- **Performance**: record flame charts, identify long tasks, see frame drops
- **Memory**: heap snapshots, allocation timeline, detect leaks
- **Network**: request waterfall (throttle to simulate slow connections)
- **Lighthouse**: audit renderer pages (limited applicability in Electron)

## Electron contentTracing API

Record Chromium trace events programmatically — the same data as `chrome://tracing`.

```ts
import { contentTracing } from 'electron';

// Start recording
await contentTracing.startRecording({
  included_categories: ['*'],  // or specific: ['v8', 'blink', 'cc']
});

// ... perform the operation you want to profile ...

// Stop and save
const path = await contentTracing.stopRecording();
// path points to a .json trace file
// Open in chrome://tracing or Perfetto UI
```

**Useful trace categories:**
- `v8` — JS execution, GC
- `blink` — rendering, layout, paint
- `cc` — compositor
- `gpu` — GPU operations
- `disabled-by-default-v8.cpu_profiler` — CPU profiling
- `disabled-by-default-memory-infra` — memory allocations

## CPU Instruction Counting

For deterministic, noise-free performance measurement (no timing jitter):

```ts
// In renderer — mark custom regions
performance.mark('component-render-start');
ReactDOM.render(<Component />, container);
performance.mark('component-render-end');
performance.measure('component-render', 'component-render-start', 'component-render-end');
```

In the trace file, `tidelta` (CPU instructions for this event) and `ticount` (total instructions at event start) provide time-agnostic measurements. Subtract start from end for exact instruction count.

**Requirements:** Chrome 78+, Linux only (not macOS), flags: `--no-sandbox --enable-thread-instruction-count`.

Use with Puppeteer for automated Lab testing:

```ts
const browser = await puppeteer.launch({
  args: ['--no-sandbox', '--enable-thread-instruction-count'],
});
const page = await browser.newPage();
await page.tracing.start({ path: 'trace.json' });
await page.goto('test.html#Button');
await page.tracing.stop();
```

One paragraph rendered in React ≈ measurably different instruction count from a span alone. Powerful for regression detection.

## Perceived Performance Metrics

Collect what matters to users:

| Metric | What it measures |
|--------|-----------------|
| Click latency | Time between click and visual update |
| Keypress latency | Time between keypress and visual update |
| Scroll latency | Time between scroll and visual update |
| TTI (Time to Interactive) | When app responds to input |
| Time to feature paint | When a specific feature renders |

Use `PerformanceObserver` and `Long Tasks API`:

```ts
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    telemetry.record('long-task', { duration: entry.duration });
  }
});
observer.observe({ entryTypes: ['longtask'] });
```

## Production Monitoring

**What Slack collects** (discovered via compiled code analysis): `timeToPageLoad`, `timeSpentInPreload`, CPU usage, memory usage, app metrics, DOM counters, trace recording controls.

**What VSCode measures:** input latency between releases to catch typing performance regressions. Startup stats dashboard comparing builds.

### Implementing Your Own

```ts
// Expose metrics from main process
app.on('ready', () => {
  const metrics = app.getAppMetrics();
  // { pid, type, cpu: { percentCPUUsage, idleWakeupsPerSecond }, memory: { ... } }
});

// In renderer
const perfData = {
  tti: performance.now(), // when interactive
  domContentLoaded: performance.timing.domContentLoadedEventEnd,
  firstPaint: performance.getEntriesByType('paint')[0]?.startTime,
};
```

## Component-Level CPU Costs

Track CPU cost per UI component using Puppeteer + trace files:

1. Render component in isolation (style guide / storybook)
2. Record trace with `page.tracing`
3. Extract `ticount` deltas from trace JSON
4. Compare before/after for each diff
5. Subtract GC events (`V8.GCScavenger` `tidelta`) for clean signal

Run on every PR for continuous regression detection. Signal is stable enough to detect the cost of adding one `<p>` tag.

**Caveat:** V8 optimizes code it runs repeatedly (JIT). First render is most expensive; subsequent renders sharply decrease. Measure consistently (always first render, or always Nth render).

## React-Specific Tools

- **React DevTools Profiler**: component render times, re-render counts
- **React Scan** (`react-scan.com`): visual overlay of re-renders
- `React.Profiler` component for programmatic measurement
- `why-did-you-render` for detecting unnecessary re-renders

> **Ref:** [Chrome Tracing](https://www.chromium.org/developers/how-tos/trace-event-profiling-tool/) · [Component CPU Costs](https://calendar.perfplanet.com/2019/javascript-component-level-cpu-costs/) · [Palette Profiling](https://palette.dev/blog/improving-performance-of-electron-apps) · [contentTracing API](https://www.electronjs.org/docs/latest/api/content-tracing)
