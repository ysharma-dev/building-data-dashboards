---
layout: chapter
title: "Chapter 23 — The PPv3 Comparison"
nav_order: 24
permalink: /23-ppv3-comparison/
---

# Chapter 23 — The PPv3 Comparison

**Skill:** segmenting one dataset into two comparable cohorts — "group A"
versus "group B" — and reusing the exact same metrics-display component
for both sides, rather than building a second bespoke view just because
it's showing "a comparison" instead of "a single view." Alongside that:
handling the very real case where one side of a comparison has no data at
all, without showing a broken or, worse, a misleadingly confident empty
chart.

## What "PPv3" means here, briefly

Recall from [Chapter 0](00-introduction.md) and
[Chapter 17](17-talking-to-the-harness-api.md): "PPv3" is a naming
convention one specific team uses for a newer generation of their
pipelines, and `isPpv3Pipeline` (built in Chapter 17) is a small classifier
function that checks whether a given pipeline's identifier or name matches
that pattern. The question this chapter answers is: **given a pile of
executions from potentially many different pipelines, how do the ones
running on PPv3 pipelines compare to everything else?** That's a
comparison view — Piece #7 from the anatomy table — and it's a shape
you'll want to reuse for any "A vs. B" question a dashboard might need to
answer, regardless of subject.

Concretely: suppose a project has six pipelines total. Two of them —
`checkout-ppv3` and `billing-ppv3` — follow the PPv3 naming convention;
the other four don't. Someone picks "All pipelines in this project" in
the filter bar you built in Chapter 20, and 140 executions come back
across all six pipelines. This component's whole job is to answer: "of
those 140, which came from a PPv3 pipeline, and how do that group's DORA
metrics compare to the other group's?" Nothing about that question is
specific to deployments — it's the same shape as "how do returning
customers compare to first-time customers" or "how does the mobile app's
error rate compare to the web app's" — which is exactly why this chapter's
skill outlives this particular app.

## Splitting one list into two, in a single pass

Open `components/ppv3-comparison.tsx`:

```tsx
"use client";

import { useMemo } from "react";
import { DoraMetricsCards } from "@/components/dora-metrics";
import { isPpv3Pipeline } from "@/lib/harness";
import type { ExecutionRecord } from "@/lib/types";

export function Ppv3Comparison({
  executions,
  pipelineCount,
}: {
  executions: ExecutionRecord[];
  pipelineCount: number;
}) {
  const { ppv3, nonPpv3 } = useMemo(() => {
    const ppv3: ExecutionRecord[] = [];
    const nonPpv3: ExecutionRecord[] = [];
    for (const e of executions) {
      const isPpv3 = isPpv3Pipeline({ identifier: e.pipelineIdentifier, name: e.pipelineName });
      (isPpv3 ? ppv3 : nonPpv3).push(e);
    }
    return { ppv3, nonPpv3 };
  }, [executions]);
```

Look closely at the loop doing the actual splitting. It's a single `for`
loop that runs once over `executions`, and for each one, decides — with
one boolean check, `isPpv3Pipeline(...)` — which of two arrays it belongs
in, then pushes it into exactly that one array: `(isPpv3 ? ppv3 : nonPpv3).push(e)`.

It would be entirely possible to write this instead as two separate calls
to `.filter()`:

```ts
const ppv3 = executions.filter((e) => isPpv3Pipeline({ identifier: e.pipelineIdentifier, name: e.pipelineName }));
const nonPpv3 = executions.filter((e) => !isPpv3Pipeline({ identifier: e.pipelineIdentifier, name: e.pipelineName }));
```

and it would produce the identical result. But notice what that costs:
`executions` gets looped over *twice*, in full, and the classification
check (`isPpv3Pipeline(...)`) runs *twice* per execution — once in each
`.filter()` call — for what is logically one decision per item ("which
bucket does this one belong to?"). The single-loop version makes exactly
one pass and calls the classifier exactly once per execution, sorting each
item into its bucket as it goes. This is a small habit worth naming
explicitly, because it comes up constantly: **whenever you're sorting
items into exactly two (or a small fixed number of) buckets based on one
test per item, a single loop with a conditional push is the more direct
tool** — not because two `.filter()` passes over a modest array would be
slow in any way you'd notice, but because one loop that makes the decision
once, per item, is more directly *what you mean*: "look at each item once,
decide where it goes." The whole thing is wrapped in `useMemo`, keyed on
`executions`, for the same reason as every other derived calculation in
this book: recompute the split only when the underlying list actually
changes, not on every unrelated re-render.

It's worth noticing that this same "one loop, one decision per item" shape
scales past two groups without changing form. If a future version of this
dashboard needed to split executions three ways — say, PPv3, a
hypothetical "PPv4," and everything else — the fix wouldn't be a third
`.filter()` call bolted on top of the first two; it would still be one
loop, just pushing each item into whichever of three arrays it belongs to
(or, cleaner still at that point, a `Map<string, ExecutionRecord[]>` keyed
by group name — the same grouping tool [Chapter 22](22-computing-metrics.md)'s
`computeMttrProxy` used to bucket executions by pipeline). The number of
buckets can grow; the "loop once, decide once per item" shape doesn't need
to change to accommodate that.

## The comparison-is-meaningless guard

Right after the split:

```tsx
  if (pipelineCount <= 1) {
    return (
      <div>
        <p>Pick "All pipelines in this project" in the Pipeline filter to compare</p>
        <p>PPv3 pipelines against everything else in this project.</p>
      </div>
    );
  }
```

Why guard on `pipelineCount <= 1` specifically, rather than just rendering
whatever the split produced regardless? Think about what a "comparison"
actually requires: two genuinely different things to compare. If the
current view is scoped to exactly one pipeline (recall from
[Chapter 20](20-building-the-filter-bar.md) that picking one specific
pipeline in the filter bar is the normal, default case), then *all* of
`executions` came from that single pipeline — meaning either `ppv3` or
`nonPpv3` will be completely empty by definition, and the other one will
be everything. That's not a comparison at all; it's just "here's the one
pipeline you already picked," dressed up in comparison-shaped UI. Showing
that would be actively confusing — someone might reasonably think the
zeroed-out side means "PPv3 has zero deployments company-wide," when it
actually just means "you didn't ask to look at more than one pipeline."

This is exactly why Chapter 20's filter bar offers the
`ALL_PIPELINES` sentinel — "All pipelines in this project" — as an
explicit option: that's the *only* selection that can put executions from
more than one pipeline into view at once, which is the only situation
where a PPv3-vs-everything-else comparison is even a meaningful question
to ask. `pipelineCount` (a prop passed down from the parent, counting how
many distinct pipelines are actually represented in `executions`) is the
signal this component uses to detect that situation, and the guard's
message tells the user exactly what to do to unlock the comparison,
rather than just silently showing nothing.

## Reusing `DoraMetricsCards` for both sides

Past the guard, the component counts how many *distinct pipelines* fall on
each side (not just how many executions — a pipeline could contribute many
executions):

```tsx
  const ppv3Pipelines = new Set(ppv3.map((e) => e.pipelineIdentifier)).size;
  const nonPpv3Pipelines = new Set(nonPpv3.map((e) => e.pipelineIdentifier)).size;
```

This is the same `Set`-for-membership tool from Chapters 21 and 22, used
here for a slightly different but related purpose: `.map()` collects every
execution's `pipelineIdentifier` (with duplicates, since one pipeline
usually has many executions), building a `new Set(...)` collapses those
duplicates down to only the distinct identifiers, and `.size` counts how
many distinct identifiers remain — a quick, idiomatic way to answer "how
many different pipelines contributed to this list?" without writing a
manual loop.

Then, the actual comparison render:

```tsx
  return (
    <div>
      {/* ... */}
      {ppv3.length > 0 ? (
        <DoraMetricsCards executions={ppv3} compact />
      ) : (
        <div>No PPv3 pipelines found in this project.</div>
      )}
      {/* mirrored for nonPpv3 */}
    </div>
  );
}
```

This is the real skill this chapter is teaching. `DoraMetricsCards` — the
component you built in [Chapter 22](22-computing-metrics.md) to render the
four DORA metric cards for *any* array of executions — gets used **twice
here, completely unchanged**, once with `ppv3` and once (mirrored, in the
part marked `{/* mirrored for nonPpv3 */}`) with `nonPpv3`. There is no
second, bespoke "comparison metrics" component anywhere in this codebase.
The only difference between the two calls is *which array of executions*
gets passed in, plus a `compact` prop that tells `DoraMetricsCards` to lay
its four cards out in a 2-column grid instead of the default 4-column one,
since two comparison panels side by side have roughly half the horizontal
room each.

This is worth sitting with, because it's easy to instinctively reach for
"the comparison view needs its own component" the first time you build one
— and that instinct is usually wrong. If a component's job is "take a list
of executions, compute metrics from them, and render cards," then *any*
subset of your data is a valid input to that same component — the whole
dataset, one pipeline's worth, PPv3's half, the other half, last month's
slice, whatever. Building it once, generically, and calling it multiple
times with different data is almost always better than writing "the normal
metrics view" and "the comparison metrics view" as two separately
maintained pieces of UI that happen to look similar today and will
inevitably drift apart over time.

## Why an empty `DoraMetricsCards` would lie

The last detail worth understanding closely is the conditional:
`ppv3.length > 0 ? <DoraMetricsCards executions={ppv3} compact /> : <div>No PPv3 pipelines found in this project.</div>`.

Why not just always render `<DoraMetricsCards executions={ppv3} compact />`,
and let it deal with an empty array if `ppv3` happens to be empty? Walk
through what `computeDoraMetrics` (from Chapter 22) would actually produce
if handed an empty array: `computeChangeFailureRate` would compute
`0 / 0`-shaped logic guarded to return `0`, so the card would confidently
display "0% Change Failure Rate." `computeDeploymentFrequency` would
likewise show "0 per day." Every number on every card would render as a
clean, specific, *confident-looking* zero.

But "0% Change Failure Rate" and "we have zero executions to measure at
all" are two completely different facts, and showing the first when the
truth is the second is actively misleading — it looks exactly like "PPv3
pipelines in this project have a flawless record," when the real situation
is "there are no PPv3 pipelines here to have a record at all." Anyone
skimming the dashboard has no way to tell those two situations apart from
the numbers alone. This is the same principle you'll see again, made even
more explicit, when [Chapter 24](24-the-option1-deep-dive.md) covers time
that's *excluded, not silently zeroed* — in both cases, the lesson is the
same: when you genuinely have no data to measure, say so directly, in
words, rather than letting an empty input quietly produce numbers that
merely *look* like a real, measured zero. That's exactly what the
`ppv3.length > 0` check buys you: a distinct, honest "No PPv3 pipelines
found in this project" message on the side that has nothing to show,
instead of a full grid of misleadingly tidy zeroes.

Notice, too, that this check has to happen on *each side independently* —
`ppv3.length > 0` guards the PPv3 panel, and a mirrored
`nonPpv3.length > 0` guards the other one. It would be a mistake to
combine them into one guard covering the whole component (something like
"only render the comparison at all if both sides have data"), because the
two sides can easily be in different states at once: it's entirely normal
for a project to have plenty of non-PPv3 executions and exactly zero PPv3
ones, especially early on while a team is migrating pipelines over to the
new convention. Guarding each side on its own data means the comparison
can show real, honest metrics for the side that has them, while still
being truthful about the side that doesn't — rather than an all-or-nothing
check hiding a perfectly good half of the comparison just because the
other half is empty.

## Checkpoint

- [ ] `components/ppv3-comparison.tsx` exists, splits `executions` into
      `ppv3` and `nonPpv3` with a single `for` loop (not two `.filter()`
      calls), memoized with `useMemo`.
- [ ] The component returns an explanatory placeholder whenever
      `pipelineCount <= 1`, instead of rendering a meaningless comparison.
- [ ] Both sides of the comparison render `DoraMetricsCards` — the exact
      same component from Chapter 22 — passing only a different
      `executions` array and a `compact` prop.
- [ ] Each side shows a distinct "no pipelines found" message instead of
      `DoraMetricsCards` when its array is empty.
- [ ] You can explain, in your own words, why showing `DoraMetricsCards`
      with a zero-length array would be misleading rather than merely
      unhelpful.

**This generalizes to:** any time you need to compare two (or more)
cohorts drawn from one dataset — this month vs. last month, Team A vs.
Team B, mobile users vs. desktop users — split the data with a single pass
over the list rather than one `.filter()` per bucket, guard against
comparisons that aren't actually meaningful yet (too little data, or only
one thing in view), and reuse your existing single-subset display
component verbatim for every cohort rather than building a second
comparison-flavored copy of it. And whenever a cohort might legitimately
be empty, render an explicit "no data" message for that cohort instead of
letting your metrics component quietly produce confident-looking zeroes
from nothing.

**This is Piece #7 from the anatomy table** in
[Chapter 0](00-introduction.md) — the Comparison View — built here not as
new metrics logic, but as a thin layer that partitions data and reuses
Chapter 22's display component twice.

Next: [Chapter 24 — The Option 1 Deep Dive](24-the-option1-deep-dive.md)
