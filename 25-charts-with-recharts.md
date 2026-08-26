---
layout: chapter
title: "Chapter 25 — Charts with Recharts"
nav_order: 26
permalink: /25-charts-with-recharts/
---

# Chapter 25 — Charts with Recharts

**Skill:** the small set of composable chart-building blocks that cover the
overwhelming majority of dashboards you'll ever build — axes, grids,
tooltips, reference lines, and legends — assembled the same way regardless
of whether you're plotting deployment durations, stock prices, or weather
readings. Once you've built the handful of chart types in this chapter,
you have essentially all the charting vocabulary most dashboards ever need.

## Why Recharts, and how it thinks

[Recharts](https://recharts.org/) is a charting library built specifically
for React: instead of calling an imperative drawing API (a "draw a line
from here to there" instruction list), you compose charts out of JSX
components — `<LineChart>`, `<XAxis>`, `<Tooltip>`, and so on — the exact
same declarative style you've been using for everything else in this book
since [Chapter 6](06-what-is-react.md). This means a chart is built,
read, and modified the same way as any other component tree: nest the
pieces you want, pass them props, and Recharts figures out the actual
pixel-level drawing.

Install it:

```bash
npm install recharts
```

## The scatter chart: every point, plotted

The simplest chart in this app is a **scatter chart** — one dot per
execution, positioned by when it happened (x-axis) and how long it took
(y-axis). Let's build `components/duration-trend-chart.tsx` piece by
piece.

First, shaping the data. Recharts expects a flat array of plain objects,
one per point, with numeric fields for whatever you're plotting:

```tsx
type Point = {
  startTs: number;
  durationMin: number;
  status: string;
  group: "before" | "after";
};

const points: Point[] = executions.map((e) => ({
  startTs: e.startTs,
  durationMin: e.durationMs / 60000,
  status: e.status,
  group: cutoffTs && e.startTs >= cutoffTs ? "after" : "before",
}));

const before = points.filter((p) => p.group === "before");
const after = points.filter((p) => p.group === "after");
```

Notice `durationMs / 60000` converts milliseconds into minutes right here,
at the data-shaping step — Recharts just draws whatever numbers you give
it; unit conversion is your job, done once, before the chart ever sees the
data. Splitting into `before`/`after` *arrays* (rather than one array with
a `group` field the chart reads) is a deliberate choice that pays off in
the next step.

Now the actual chart:

```tsx
<ResponsiveContainer width="100%" height="100%">
  <ScatterChart margin={{ top: 8, right: 16, bottom: 8, left: 24 }}>
    <CartesianGrid stroke="var(--chart-grid)" strokeDasharray="0" vertical={false} />
    <XAxis
      dataKey="startTs"
      type="number"
      domain={["dataMin", "dataMax"]}
      tickFormatter={formatDate}
    />
    <YAxis
      dataKey="durationMin"
      label={{ value: "Duration (minutes)", angle: -90, position: "left" }}
    />
    <Tooltip content={<CustomTooltip />} />
    <Legend formatter={(value) => (value === "before" ? "Before rollout" : "After rollout")} />
    <Scatter name="before" data={before} fill="var(--chart-before)" />
    <Scatter name="after" data={after} fill="var(--chart-after)" />
  </ScatterChart>
</ResponsiveContainer>
```

Walking through each building block, since every chart type in this
chapter reuses most of these same pieces:

- **`<ResponsiveContainer>`** wraps the whole chart and makes it resize to
  fill whatever space its parent element gives it — `width="100%"
  height="100%"` means "however big your container element is, that's how
  big I'll be." Wrap the outer container in a plain `<div>` with a fixed
  height (like `className="h-[360px] w-full"`) so the chart has *some*
  concrete size to fill.
- **`<CartesianGrid>`** draws the faint background gridlines. `vertical={false}`
  disables the vertical gridlines, keeping just the horizontal ones — a
  common, less-visually-busy choice for this kind of chart.
- **`<XAxis>`/`<YAxis>`** — each needs a `dataKey` telling it which field
  of each point object to read. `type="number"` on the x-axis (since
  `startTs` is a timestamp, a plain number, not a category label) plus
  `domain={["dataMin", "dataMax"]}` tells Recharts to size the axis to
  exactly span your actual data's range, rather than starting at zero.
  `tickFormatter` lets you supply a function converting a raw value (a
  timestamp) into a human-readable label (a short date string) for display
  only — the underlying data stays numeric.
- **`<Tooltip>`** — the little box that appears when you hover over a
  point. `content={<CustomTooltip />}` swaps Recharts' default tooltip for
  your own component, which receives the hovered point's data and can
  render it however you like (we'll look at a `CustomTooltip` shortly).
- **`<Legend>`** — the small key at the bottom explaining what each color
  means. `formatter` lets you turn an internal series name (`"before"`)
  into a nicer human-facing label ("Before rollout") without changing the
  underlying data.
- **`<Scatter>`** — the actual data series. This is *why* we split `points`
  into two separate arrays earlier: two separate `<Scatter>` elements, each
  with its own `data` and its own `fill` color, is how Recharts renders two
  differently-colored groups of dots on the same chart. If you'd kept one
  array and tried to color individual points conditionally, you'd need a
  much more awkward per-point styling approach — splitting into two series
  up front is simpler and is the idiomatic Recharts way to show two
  categories on one chart.

## A custom tooltip is just a component

`CustomTooltip` is worth building because Recharts' default tooltip is
generic — showing your own, tailored to exactly the fields you care about,
makes a chart feel purpose-built rather than assembled off a template:

```tsx
function CustomTooltip({ active, payload }: { active?: boolean; payload?: { payload: Point }[] }) {
  if (!active || !payload || payload.length === 0) return null;
  const p = payload[0].payload;
  return (
    <div className="rounded-md border border-border bg-popover px-3 py-2 text-xs shadow-md">
      <div className="font-medium text-popover-foreground">{new Date(p.startTs).toLocaleString()}</div>
      <div className="text-muted-foreground">Duration: {formatDuration(p.durationMin * 60000)}</div>
      <div className="text-muted-foreground">Status: {p.status}</div>
    </div>
  );
}
```

Recharts calls this component itself, passing `active` (is the tooltip
currently visible at all — `false` when the mouse isn't hovering
anything) and `payload` (an array — usually with one entry per data series
under the cursor — where `payload[0].payload` is the *original* point
object you gave the `<Scatter>` element). The `if (!active || ...) return
null;` guard is essential: without it, this component would try to render
even when nothing is being hovered, and `payload[0]` would fail since the
array would be empty.

## A reference line, for a specific moment in time

The rollout cutoff date deserves a visual marker directly on the chart —
Recharts calls this a `<ReferenceLine>`:

```tsx
{cutoffTs && (
  <ReferenceLine
    x={cutoffTs}
    stroke="var(--chart-ref-line)"
    strokeDasharray="4 4"
    label={{ value: "Rollout", position: "insideTopLeft" }}
  />
)}
```

`x={cutoffTs}` draws a vertical line at that exact x-axis position (since
`cutoffTs` is measured in the same units — a timestamp — as the `startTs`
values the x-axis is plotting). `strokeDasharray="4 4"` makes it dashed
rather than solid, a common convention for "this line is an annotation, not
real data." Wrapping the whole element in `{cutoffTs && (...)}` means it
simply doesn't render at all when no cutoff date has been chosen — you saw
this exact conditional-rendering pattern back in
[Chapter 6](06-what-is-react.md).

## A bar chart: categories, not time

The skip-rate chart in the Option 1 deep dive
([Chapter 24](24-the-option1-deep-dive.md)) is a **bar chart** — comparing
a handful of *categories* (pipeline/tier combinations) against each other,
rather than plotting individual points over time. The building blocks are
almost identical, with a few notable differences:

```tsx
<BarChart data={chartSummaries} layout="vertical" margin={{ top: 8, right: 24, bottom: 8, left: 8 }}>
  <CartesianGrid horizontal={false} />
  <XAxis type="number" unit="%" domain={[0, 100]} />
  <YAxis type="category" dataKey="key" width={220} />
  <Tooltip content={<ChartTooltip />} />
  <Bar dataKey="skipRate" name="skipRate" fill="var(--chart-before)" radius={[0, 4, 4, 0]} />
</BarChart>
```

- **`layout="vertical"`** here means the *bars* are horizontal (a slightly
  confusing naming choice in Recharts — "vertical" refers to the
  categories being stacked vertically, one row per bar, rather than the
  bars themselves being upright columns). This is a deliberate choice for
  a list of pipeline/tier names, which tend to be long — horizontal bars
  give each label plenty of horizontal room to read, whereas upright
  columns would force long names to be squeezed or rotated.
- **`<XAxis type="number" unit="%" domain={[0, 100]}>`** fixes the scale
  to exactly 0-100, since we know in advance every skip rate is a
  percentage — there's no reason to let Recharts auto-scale to
  "whatever the data happens to range over" when you already know the
  meaningful, fixed bounds.
- **`<YAxis type="category" dataKey="key">`** — this axis holds category
  *labels* (pipeline/tier names), not numbers, so `type="category"`
  instead of `type="number"`.
- **`radius={[0, 4, 4, 0]}`** rounds only the *outer* corners of each bar
  (top-right and bottom-right, for a horizontal bar growing rightward) — a
  small polish detail; the four numbers correspond to each of a bar's four
  corners.

## A line chart: a smoothed trend

Finally, the "deployment time trend" chart from
[Chapter 24](24-the-option1-deep-dive.md) — a single continuous line,
rather than a scatter of individual dots, showing the *trend* rather than
every individual data point:

```tsx
<LineChart data={points}>
  {/* ...same CartesianGrid, XAxis, YAxis, Tooltip, ReferenceLine pattern as above... */}
  <Line
    type="monotone"
    dataKey="avgDurationMin"
    stroke="var(--chart-after)"
    strokeWidth={2}
    dot={{ r: 3 }}
  />
</LineChart>
```

The chart-building blocks (`CartesianGrid`, `XAxis`, `YAxis`, `Tooltip`,
`ReferenceLine`) are identical to the scatter chart above — this is the
whole point of a composable charting library: learn the pieces once, and
mostly just swap which "series" component (`<Scatter>`, `<Bar>`, `<Line>`)
you nest them around. `type="monotone"` tells Recharts to draw a smoothly
curved line between points rather than sharp straight-line segments — a
purely visual choice.

The real work for this chart happens *before* it's ever rendered, in how
the data gets pre-processed: rather than plotting every single raw
execution (which would just be noisy, hard to read as a "trend"), the data
is smoothed first — either bucketed into weekly averages (if there's
enough history to make weekly buckets meaningful) or averaged over a
rolling window of the last several executions (if the history is too
short for weekly buckets to make sense). This is a genuinely useful
general lesson: often, the most important part of a "trend" chart isn't
the charting library at all — it's the plain data-transformation code that
runs *before* the chart ever sees the data, deciding what "smoothed"
actually means for your specific data's shape and volume.

## Checkpoint

- [ ] `npm install recharts` has been run.
- [ ] You've built at least one chart using `<ResponsiveContainer>`,
      `<CartesianGrid>`, `<XAxis>`, `<YAxis>`, and a custom `<Tooltip>`
      component.
- [ ] You can explain why the scatter chart splits its data into two
      separate arrays (`before`/`after`) rather than one array with a
      `group` field.
- [ ] You can explain the difference between plotting every raw data point
      (a scatter chart) versus plotting a smoothed trend (a line chart
      over pre-averaged data).

**This generalizes to:** these same five or six building blocks —
grid, axes, tooltip, legend, reference line, and a choice of series type
(`Scatter`/`Bar`/`Line`, or Recharts' `Area` and `Pie` which follow the
identical composition pattern) — cover the vast majority of charts any
dashboard will ever need, regardless of subject. Whatever you're plotting
next, the real design decisions are usually about the *data* going into the
chart (what to smooth, what to split into series, what scale to fix versus
auto-size) far more than about Recharts' API itself.

**This is Piece #9 from the anatomy table** in
[Chapter 0](00-introduction.md) — Charts.

Next: [Chapter 26 — Exporting a PDF Report](26-exporting-a-pdf-report.md)
