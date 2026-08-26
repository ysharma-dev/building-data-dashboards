---
layout: chapter
title: "Chapter 24 — The Option 1 Deep Dive"
nav_order: 25
permalink: /24-the-option1-deep-dive/
---

# Chapter 24 — The Option 1 Deep Dive

**Skill:** building a "deep dive" analysis view — a more detailed, focused
investigation of one specific question, layered on top of the basics
you've already built (filtering, fetching, metrics). Specifically, this
chapter teaches three things that generalize far beyond this app: scoping
an analysis to a date range without breaking the rest of your data, fetching
and joining data from a *second* source to answer a question your first
data source can't answer alone, and correctly removing a confound (from
[Chapter 12](12-the-option1-optimization-story.md)) using real interval
math rather than simple subtraction.

This is the single most complex component in the whole app —
`components/option1-analysis.tsx`. We'll build it in stages, exactly the
way you'd actually approach a feature this size: get the basic shape
working, then layer in the harder pieces one at a time.

## The rollout cutoff date, and why it needs its own state

Recall from [Chapter 12](12-the-option1-optimization-story.md): the reader
might want to answer "how much did Option 1 help *since it was rolled
out*," which means excluding older executions from before the optimization
even existed. That requires a **date picker**, independent of anything in
the top-level filter bar from [Chapter 20](20-building-the-filter-bar.md):

```tsx
const [cutoffDate, setCutoffDate] = useState("");
const cutoffTs = cutoffDate ? new Date(cutoffDate).getTime() : null;
```

`cutoffDate` is the raw string from an `<input type="date">` (which
browsers natively render as a real date picker, no library needed).
`cutoffTs` converts that string into a timestamp — the same
milliseconds-since-1970 representation used by every `startTs`/`endTs`
field on an `ExecutionRecord` (recall [Chapter 16](16-defining-types.md))
— or `null` if no date has been chosen. Every calculation below compares
against `cutoffTs`, a number, never against the raw date string.

## The double-filtering trap

Here's a genuinely important lesson, and one this project actually got
wrong once during real development — worth understanding *why* it's a
trap, not just *that* it's one.

The Option 1 skip-rate analysis (how often the optimization fired) should
scope to only executions from the cutoff date forward — "analyze the
rollout, not everything before it existed." That's a deliberate choice:

```tsx
const executionsForSkipAnalysis = useMemo(() => {
  if (!cutoffTs) return executions;
  return executions.filter((e) => e.startTs >= cutoffTs);
}, [executions, cutoffTs]);
```

But the duration charts and summary cards further down the page need to
show a **before/after comparison** — they need to see executions on
*both* sides of the cutoff, so they can render "before" data next to
"after" data. If you fed `executionsForSkipAnalysis` (already filtered to
*only* post-cutoff executions) into a component that itself tries to split
its input into "before" and "after" groups, there would be nothing left
to put in the "before" group — every execution it received would already
be post-cutoff. The chart would render, but it would silently show only
one color, looking broken without throwing any error.

The fix: keep two separate ideas distinct. `executionsForSkipAnalysis` is
scoped for one specific purpose (the skip-rate table); the raw, full
`executions` array (unfiltered) gets passed to anything that needs to do
its *own* before/after split. Passing an already-filtered array into a
component that expects to do its own filtering is the trap — always be
clear about *whether* a piece of data has already been filtered before
handing it to something that filters again.

## Discovering which executions even apply

Not every execution has a child-pipeline link at all — recall
[Chapter 17](17-talking-to-the-harness-api.md)'s `childStageLinks` field,
which is often an empty array. We turn every execution's links into a flat
list of "things to go fetch," using `.flatMap()` (introduced briefly in
[Chapter 4](04-javascript-crash-course.md); it's like `.map()`, except when
your mapping function itself returns an array, `.flatMap()` flattens all
those little arrays into one big one instead of leaving you with an array
of arrays):

```tsx
const batchItems = useMemo(() => {
  return executionsForSkipAnalysis.flatMap((e) =>
    e.childStageLinks.map((link) => ({
      stageName: link.stageName,
      parentExecutionId: e.id,
      pipelineName: e.pipelineName,
      childOrg: link.childOrgId,
      childProject: link.childProjectId,
      childPlanExecutionId: link.childPlanExecutionId,
    })),
  );
}, [executionsForSkipAnalysis]);
```

This is exactly the shape the batch API route from
[Chapter 19](19-remaining-api-routes.md) expects as its request body. And
because of the double-filtering lesson above, we need a *second*, parallel
version of this same transformation — built from the *unfiltered*
`executions`, used only to compute adjusted durations that need to see both
sides of the cutoff:

```tsx
const allExecutionsBatchItems = useMemo(() => {
  return executions.flatMap((e) =>
    e.childStageLinks.map((link) => ({
      stageName: link.stageName,
      parentExecutionId: e.id,
      pipelineName: e.pipelineName,
      childOrg: link.childOrgId,
      childProject: link.childProjectId,
      childPlanExecutionId: link.childPlanExecutionId,
    })),
  );
}, [executions]);
```

Two lists, two independent purposes, feeding two independent fetches.

## Fetching from a second data source, twice, independently

Both lists get sent to the batch endpoint, each tracked with its own
`FetchState` — reusing exactly the loading/error/data three-state pattern
from [Chapter 21](21-fetching-and-displaying-executions.md):

```tsx
type FetchState =
  | { kind: "loading"; total: number }
  | { kind: "error"; message: string }
  | { kind: "done"; outcomes: (OptimizationOutcome & { pipelineName: string })[] };

async function fetchOptimizationBatch(batchItems) {
  const res = await fetch("/api/optimization/batch", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(batchItems),
  });
  const data = await res.json();
  if (!res.ok) throw new Error(data.error ?? "Failed to load optimization data");
  if (!Array.isArray(data)) throw new Error("Unexpected response shape");

  const nameByChild = new Map(batchItems.map((b) => [b.childPlanExecutionId, b.pipelineName]));
  return data.map((o) => ({
    ...o,
    pipelineName: nameByChild.get(o.childPlanExecutionId) ?? "Unknown pipeline",
  }));
}
```

Notice the `nameByChild` Map at the end. Recall from
[Chapter 19](19-remaining-api-routes.md) that the batch API route's
response doesn't include a `pipelineName` field — the underlying
`OptimizationOutcome` type from [Chapter 16](16-defining-types.md) simply
doesn't carry one, since `getChildStageOutcome` has no reason to know it.
So this function *re-attaches* the pipeline name on the client side,
matching each returned outcome back to the request item it came from via
`childPlanExecutionId` (a value both the request and the response share).
This is a small but genuinely useful pattern: when a response is missing a
field you need for display purposes, and you already had that field in
the request that produced it, join them back together locally rather than
changing the API to carry data it doesn't otherwise need.

Both fetches are triggered by their own `useEffect`, with a `requestId`
ref to guard against a subtle race (covered fully in
[Chapter 27](27-polish-accessibility-and-bugfixes.md) — for now, just
notice that each effect checks its own request counter before committing
its result, so a fast-changing filter selection can't let a slow, stale
fetch overwrite a newer one's result).

## Summarizing outcomes into skip-rate tiers

Once outcomes have loaded, they need to be grouped and summarized. The
grouping key depends on context — if you're looking at one specific
pipeline, group by *stage name* alone (since there's only one pipeline in
view); if you're looking at "all pipelines in this project," group by
*pipeline name + stage name* together, so tiers from different pipelines
don't get merged into one misleading average:

```tsx
function summarize(outcomes, groupByPipeline: boolean) {
  const byGroup = new Map<string, typeof outcomes>();
  for (const o of outcomes) {
    const key = groupByPipeline ? `${o.pipelineName} · ${o.stageName}` : o.stageName;
    const list = byGroup.get(key) ?? [];
    list.push(o);
    byGroup.set(key, list);
  }

  let excludedGroupCount = 0;

  const summaries = Array.from(byGroup.entries())
    .map(([key, list]) => {
      const applicableList = list.filter(isApplicable);
      const skipped = applicableList.filter((o) => o.optimizationFired).length;
      // ...averages, time-saved, and the two wait-time averages computed here
      return { key, /* ...totals... */ };
    })
    .filter((s) => {
      if (s.applicable === 0) {
        excludedGroupCount += 1;
        return false;
      }
      return true;
    })
    .sort((a, b) => b.total - a.total);

  return { summaries, excludedGroupCount };
}
```

Two things worth real attention:

- **`isApplicable`** filters to only outcomes where the skip-check step was
  genuinely present at all (some child pipeline templates don't have one).
  Averaging skip rate over executions that never had a skip *check* in the
  first place would be meaningless — you can't measure whether something
  fired if the mechanism to fire wasn't even there.
- **The final `.filter()` step, with its `excludedGroupCount` counter**, is
  doing exactly the lesson from [Chapter 23](23-ppv3-comparison.md): a
  group with **zero applicable samples** gets *dropped entirely*, rather
  than shown as a misleading "0% skip rate." Zero applicable samples means
  "we have no measurement," not "we measured, and it was zero" — those are
  different facts, and showing the first as though it were the second
  would be actively misleading. The dropped count is still tracked and
  shown to the user as a small disclosure ("N groups had no applicable
  samples and are omitted"), so nothing silently vanishes without a trace.

## The confound: isolating approval wait time

Now the part [Chapter 12](12-the-option1-optimization-story.md) set up:
the Approve Deploy and Validate Clusters wait times must never be folded
into the skip-rate or time-saved numbers. Inside the same `summarize`
function, they're computed completely separately, over the *whole* group
(not just `applicableList`, since these gates exist independently of
whether the skip-check step is even present):

```tsx
const approveDeploy = averageDuration(list.map((o) => o.approveDeployDurationMs));
const validateClusters = averageDuration(list.map((o) => o.validateClustersDurationMs));
```

where:

```tsx
function averageDuration(durations: (number | null)[]) {
  const present = durations.filter((d): d is number => d !== null);
  if (present.length === 0) return { avg: null, count: 0 };
  return { avg: present.reduce((a, b) => a + b, 0) / present.length, count: present.length };
}
```

Notice `averageDuration` filters out `null`s *before* averaging, and
reports both the average *and* the count of real samples it was computed
from — recall [Chapter 16](16-defining-types.md)'s lesson: a `null` here
means "this gate didn't run for this execution" (maybe it was skipped, or
doesn't apply to this pipeline), and averaging over a `null` as though it
were `0` would silently and wrongly drag the average down toward zero,
making the wait time look smaller than it really is whenever it *does*
occur.

## Merging overlapping wait windows: the interval-math problem

Here's the trickiest, most instructive part of this entire chapter. A
single parent execution can have *multiple* tiers, each with its own
Approve Deploy and Validate Clusters wait window. If two tiers happen to
be waiting on the very same underlying approval gate at nearly the same
time (a realistic scenario — one Jira ticket approval can gate several
tiers deploying together), and you naively **sum** each tier's own wait
duration, you double-count the same wall-clock time twice.

The fix is a small, genuinely reusable utility — `lib/interval-merge.ts`:

```ts
export type TimeInterval = { start: number; end: number };

export function mergedDurationMs(intervals: TimeInterval[]): number {
  if (intervals.length === 0) return 0;

  const sorted = [...intervals].sort((a, b) => a.start - b.start);
  let totalMs = 0;
  let currentStart = sorted[0].start;
  let currentEnd = sorted[0].end;

  for (let i = 1; i < sorted.length; i++) {
    const next = sorted[i];
    if (next.start <= currentEnd) {
      currentEnd = Math.max(currentEnd, next.end);
    } else {
      totalMs += currentEnd - currentStart;
      currentStart = next.start;
      currentEnd = next.end;
    }
  }

  totalMs += currentEnd - currentStart;
  return totalMs;
}
```

Walk through this slowly, since the algorithm itself is a classic,
widely-useful one (not specific to Harness or deployments at all — it's
the general "merge overlapping intervals" problem):

1. **Sort every interval by its start time.** Once sorted, you only ever
   need to compare each interval against the *single currently-open merged
   range*, never against every other interval — a huge simplification.
2. **Walk through the sorted intervals, tracking one "current" open
   range** (`currentStart`, `currentEnd`).
3. **If the next interval starts before (or exactly when) the current
   range ends** (`next.start <= currentEnd`), it overlaps or touches the
   current range — extend `currentEnd` to whichever is later, the current
   range's own end or this interval's end (`Math.max(...)`, in case the
   new interval is fully contained inside the current one and wouldn't
   actually extend it).
4. **If the next interval starts strictly after the current range ends**,
   there's a genuine gap — the current merged range is finished. Add its
   length (`currentEnd - currentStart`) to the running `totalMs`, then
   start a brand-new "current" range from this next interval.
5. **After the loop, add the final open range's length too** — it's easy
   to forget this last step, since the loop only closes out a range when
   it finds a *gap* after it; the very last range never gets a "next"
   interval to trigger that, so it has to be added explicitly once the
   loop ends.

The result: total wall-clock time actually spent waiting, with any overlap
between concurrent tiers counted exactly once — not twice, not averaged
away, just correctly merged.

## Applying the merge to adjust durations

With `mergedDurationMs` available, we can now compute "what would this
execution's duration have been, if we hadn't counted approval/validation
wait time at all":

```tsx
const adjustedExecutions = useMemo(() => {
  if (allExecutionsFetchState?.kind !== "done") return [];

  const intervalsByParent = new Map<string, TimeInterval[]>();
  for (const o of allExecutionsFetchState.outcomes) {
    const intervals = intervalsByParent.get(o.parentExecutionId) ?? [];
    if (o.approveDeployStartTs !== null && o.approveDeployEndTs !== null) {
      intervals.push({ start: o.approveDeployStartTs, end: o.approveDeployEndTs });
    }
    if (o.validateClustersStartTs !== null && o.validateClustersEndTs !== null) {
      intervals.push({ start: o.validateClustersStartTs, end: o.validateClustersEndTs });
    }
    intervalsByParent.set(o.parentExecutionId, intervals);
  }

  return executions.map((e) => {
    const intervals = intervalsByParent.get(e.id);
    if (!intervals || intervals.length === 0) return e;

    const waitMs = mergedDurationMs(intervals);
    return { ...e, durationMs: Math.max(e.durationMs - waitMs, 0) };
  });
}, [allExecutionsFetchState, executions]);
```

Notice this builds a Map from each *parent execution's* ID to *all* the
wait intervals collected across *every one of its tiers* — this is exactly
the scenario `mergedDurationMs` was written for. Then, for every
execution, it subtracts the merged (not summed) wait time from the raw
duration, clamped at zero with `Math.max(..., 0)` as a defensive floor
(durations should never go negative, even if timing data were ever
slightly inconsistent). Executions with no child links at all pass through
completely unchanged — there's nothing to adjust.

This `adjustedExecutions` array — built from the *unfiltered* `executions`,
recall the double-filtering lesson from earlier in this chapter — is what
gets handed to the `SummaryCards`, duration charts, and execution table
further down the page, so every duration number they show already has
approval/validation wait time correctly excluded.

## Showing the excluded time, not hiding it

Finally, the confound doesn't just get subtracted and forgotten — it's
displayed in its own clearly-labeled panel, with an explicit badge reading
"excluded from the numbers above," plus a plain-language explanation of
*why*. This directly closes the loop on [Chapter 12](12-the-option1-optimization-story.md)'s
lesson: isolating a confound isn't the same as hiding it. The reader
should be able to see exactly how much time was excluded and why, not just
trust that "the numbers are adjusted somehow."

## Checkpoint

- [ ] `lib/interval-merge.ts` exists and exports `mergedDurationMs`.
- [ ] `components/option1-analysis.tsx` maintains two independent
      `FetchState`s — one for the cutoff-scoped skip-rate analysis, one for
      the unfiltered, full-execution-set duration adjustment.
- [ ] Setting a rollout cutoff date still shows both "before" and "after"
      data in the duration charts (not just one color) — this is the
      double-filtering bug, fixed.
- [ ] The "Approval & validation wait time" panel shows a non-zero value
      when child pipelines have real approval gates in your data, with an
      "excluded from the numbers above" badge.

**This generalizes to:** any "deep dive" feature you build later — a
detailed investigation into one specific question, layered on top of a
dashboard's basic filtering and metrics — will likely need this same three
-part toolkit: an independent date-range or scope control that doesn't
accidentally re-filter data that's already been filtered elsewhere; a
second, joined data source when your first one can't answer the deep
question alone; and, whenever your investigation involves any kind of time
window, the general interval-merge algorithm for correctly combining
overlapping ranges — a problem that reappears anywhere concurrent things
can share time (overlapping meeting schedules, overlapping server outages,
overlapping shifts), not just approval gates.

**This is Piece #8 from the anatomy table** in
[Chapter 0](00-introduction.md) — the Deep-Dive View. In
[Chapter 29](29-building-your-own-dashboard.md), you'll sketch this exact
piece for a different subject entirely.

Next: [Chapter 25 — Charts with Recharts](25-charts-with-recharts.md)
