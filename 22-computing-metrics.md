# Chapter 22 — Computing Metrics

**Skill:** turning a list of raw records into summary metrics — as actual,
written code. [Chapter 11](11-metrics-explained.md) already taught you the
conceptual shapes every metric falls into (a rate, a ratio, or an honest
proxy) and walked through the exact DORA formulas and the real subtleties
hiding underneath two of them. This chapter doesn't re-derive any of that
— it's about the **engineering**: how to structure the code that computes
several related metrics, so it stays readable, testable, and easy to add a
fifth metric to later. This is **Piece #6** from the anatomy table in
[Chapter 0](00-introduction.md), the Metrics Module, built for real.

## One pure function per metric

Open `lib/dora.ts`. The very first design decision is structural: instead
of one giant function that computes all four DORA metrics inline, each
metric gets its own small function, and none of those four functions is
exported:

```ts
import type { DoraMetrics, ExecutionRecord } from "./types";

const DAY_MS = 24 * 60 * 60 * 1000;
const FAILURE_STATUSES = new Set(["Failed", "Aborted"]);

function computeDeploymentFrequency(executions: ExecutionRecord[]) {
  const successCount = executions.filter((e) => e.status === "Success").length;
  if (executions.length === 0) {
    return { perDay: 0, successCount: 0, windowDays: 0 };
  }
  const timestamps = executions.map((e) => e.startTs);
  const spanMs = Math.max(...timestamps) - Math.min(...timestamps);
  const windowDays = Math.max(spanMs / DAY_MS, 1);
  return { perDay: successCount / windowDays, successCount, windowDays };
}
```

(This is the exact Deployment Frequency formula from Chapter 11 — the
`Math.max(..., 1)` floor against a zero-day span is the same defensive
trick discussed there. If any of the *math* here is unfamiliar, that's
your cue to flip back to Chapter 11; this chapter assumes you already
understand *why* each formula works and focuses only on how it's written.)

Each of the other three metrics — `computeChangeFailureRate`,
`computeLeadTimeProxy`, `computeMttrProxy` — gets its own function the
same way, each taking `executions: ExecutionRecord[]` and returning its own
small result object. None of these four functions has the keyword
`export` in front of it. Only one function at the bottom of the file does:

```ts
export function computeDoraMetrics(executions: ExecutionRecord[]): DoraMetrics {
  return {
    deploymentFrequency: computeDeploymentFrequency(executions),
    changeFailureRate: computeChangeFailureRate(executions),
    leadTimeProxyMs: computeLeadTimeProxy(executions),
    mttrProxyMs: computeMttrProxy(executions),
  };
}
```

Why structure it this way, rather than one long exported function with all
four calculations inline? Two reasons, both worth internalizing as habits
you'll reuse constantly:

- **Readability.** A function named `computeChangeFailureRate` that does
  exactly one thing is something you can read, understand, and verify in
  isolation — you don't have to hold four unrelated calculations in your
  head at once to check that any single one is correct.
- **Encapsulation.** `computeDoraMetrics` is the *only* function anything
  outside this file ever needs to know about. Every caller — the component
  you'll see below, a test you might write, code six months from now —
  gets one clean object back and never needs to know, or care, that the
  four numbers inside it were computed by four separate functions. If you
  later decide to compute Change Failure Rate differently, or add a fifth
  metric, every caller of `computeDoraMetrics` keeps working unchanged,
  because the *shape* of what's exported hasn't moved. This is the same
  "isolate the knowledge in one place" idea from Chapter 17's
  `lib/harness.ts` — there, one file was the only place that knew how the
  Harness API worked; here, one function is the only place that knows how
  many metrics exist or how they're individually computed.

## Small shared helpers instead of duplicated logic

Two of the four metric functions need the same basic statistics — an
average, and a median (the middle value of a sorted list). Rather than
writing that math twice, two tiny private helpers sit at the top of the
file, used by whichever metric functions need them:

```ts
function median(values: number[]): number | null {
  if (values.length === 0) return null;
  const sorted = [...values].sort((a, b) => a - b);
  const mid = Math.floor(sorted.length / 2);
  return sorted.length % 2 === 0 ? (sorted[mid - 1] + sorted[mid]) / 2 : sorted[mid];
}

function average(values: number[]): number | null {
  if (values.length === 0) return null;
  return values.reduce((a, b) => a + b, 0) / values.length;
}
```

Both return `null` rather than `0` or `NaN` when `values` is empty — the
same "honestly represent 'we don't know'" habit from `OptimizationOutcome`
in [Chapter 16](16-defining-types.md). An average of zero numbers isn't
meaningfully zero; it's undefined, and `null` says that plainly rather
than lying with a number that looks real. `median` sorts a *copy* of
`values` (`[...values]`, never the original array — the same
never-mutate-what-you-were-handed discipline from
[Chapter 21](21-fetching-and-displaying-executions.md)'s `sorted` table
rows) and then picks either the single middle element (odd length) or the
average of the two middle elements (even length) — standard median
arithmetic, written once, here, instead of copied into every function that
needs it.

This is a small but important instinct: the moment you notice two
different metric functions need the same underlying calculation, pull it
out into its own tiny, named function immediately. It's easier to trust
`median()` is correct once, in one place, than to trust that two
copy-pasted inline calculations both got the "even vs. odd length" logic
right independently.

## One `Set`, defined once, used by two functions

```ts
const FAILURE_STATUSES = new Set(["Failed", "Aborted"]);
```

This line sits at module scope — outside any function, computed exactly
once when this file first loads — and gets reused by two different metric
functions: `computeChangeFailureRate` (counting how many executions have a
status in this set) and `computeMttrProxy` (finding which executions count
as a "failure" to look for a subsequent recovery from). Defining it once,
by name, means both functions are guaranteed to agree on exactly which
statuses count as a failure — if you needed to add, say, `"Expired"` to the
list of failure statuses tomorrow, there's exactly one line to change, and
both metrics update consistently.

Why a `Set` here rather than a plain array checked with `.includes()`?
Exactly the same reasoning as `expanded` in Chapter 21's execution table:
the question being asked, over and over, once per execution, is "is this
one status *a member of* this small fixed collection?" — and a `Set`'s
`.has()` is the tool built specifically for that membership question. An
array with `.includes()` would work too, functionally, but reaching for a
`Set` signals — to you, later, and to anyone else reading this file —
exactly what kind of check this is: membership, not sequence or order.

## Grouping by key with a Map

`computeMttrProxy` needs to look at each pipeline's executions
*separately* (recall Subtlety A from Chapter 11 — you must never compare a
failure on one pipeline against a success on a completely unrelated one).
That means the first thing this function has to do is **group** a flat
list of executions into buckets, one bucket per pipeline:

```ts
function computeMttrProxy(executions: ExecutionRecord[]) {
  const byPipeline = new Map<string, ExecutionRecord[]>();
  for (const e of executions) {
    const list = byPipeline.get(e.pipelineIdentifier) ?? [];
    list.push(e);
    byPipeline.set(e.pipelineIdentifier, list);
  }
  const recoveryTimes: number[] = [];
  for (const pipelineExecutions of byPipeline.values()) {
    const sorted = [...pipelineExecutions].sort((a, b) => a.startTs - b.startTs);
    for (let i = 0; i < sorted.length; i++) {
      if (!FAILURE_STATUSES.has(sorted[i].status)) continue;
      const nextSuccess = sorted.slice(i + 1).find((e) => e.status === "Success" && e.startTs >= sorted[i].endTs);
      if (nextSuccess) recoveryTimes.push(nextSuccess.startTs - sorted[i].endTs);
    }
  }
  return { averageMs: average(recoveryTimes), sampleCount: recoveryTimes.length };
}
```

Focus on the grouping loop first — it's a pattern you'll use constantly,
anywhere you need to bucket a flat list by some key:

```ts
const list = byPipeline.get(e.pipelineIdentifier) ?? [];
list.push(e);
byPipeline.set(e.pipelineIdentifier, list);
```

Read it as three steps, repeated for every execution: **get** whichever
array is currently stored under this pipeline's identifier — or, if
nothing's been stored there yet (this is the *first* execution seen for
this pipeline), fall back to a fresh empty array via `?? []`; **mutate**
that array by pushing the current execution onto the end of it; then
**put it back** into the map under the same key with `.set()`. The very
first time a given `pipelineIdentifier` is seen, `.get()` returns
`undefined`, `?? []` supplies a brand-new empty array, and `.set()` stores
it for the first time. Every subsequent execution for that same pipeline
finds the existing array via `.get()`, appends to it, and stores it right
back. By the end of the loop, `byPipeline` maps each pipeline identifier
to exactly the executions that belong to it — a `Map<string, ExecutionRecord[]>`,
chosen (over a plain object) because the keys here are arbitrary runtime
strings (pipeline identifiers you don't know in advance), which is exactly
what `Map` is designed for. This "get with a fallback, mutate, put it
back" shape recurs anywhere you need to group a list by some field — group
orders by customer, group log lines by request ID, group expenses by
category — it's always the same three steps.

The nested loop after that — walking each pipeline's own executions in
time order, checking each failure against later executions on that same
pipeline, requiring `nextSuccess.startTs >= sorted[i].endTs` — is exactly
the algorithm Chapter 11 walked through in detail (Subtlety A: grouping
before comparing; Subtlety B: comparing against `endTs`, not `startTs`, to
avoid the negative-recovery-time bug from overlapping executions). The
*why* behind that `>=` comparison was already covered thoroughly there —
this chapter is only about the code's shape, so if any of that logic feels
unfamiliar, that's your cue to go back and reread Chapter 11's worked
example rather than re-deriving it here.

## The display side: `DoraMetricsCards`

With `computeDoraMetrics` written, `components/dora-metrics.tsx` is what
actually puts it on screen. Conceptually, its top-level component looks
like this:

```tsx
export function DoraMetricsCards({ executions, compact }: { executions: ExecutionRecord[]; compact?: boolean }) {
  const metrics = computeDoraMetrics(executions);
  // ...renders one MetricCard per metric, in a compact ? 2-column : 4-column grid
}
```

Notice `computeDoraMetrics(executions)` is called **directly in the
component body** — not inside a `useEffect`, and with no `loading` state
anywhere nearby. That's a deliberate contrast with
[Chapter 21](21-fetching-and-displaying-executions.md)'s fetch logic: a
network request is *asynchronous* (you don't have the data yet, and won't
for some unknown amount of time, so you need `loading`/`error`/`data`
tracked as separate state), but `computeDoraMetrics` is a **synchronous,
pure calculation** over data that's *already* sitting in memory by the
time this component renders. There's nothing to wait for and nothing that
can fail over the network — it's arithmetic over an array you already
have, so it runs straight through, every render, with no state and no
effect involved at all. Recognizing which of your code is "waiting on the
outside world" (needs the three-state pattern) versus "just computing
something from data you already hold" (doesn't) is a distinction worth
making consciously every time, rather than reflexively wrapping every
calculation in an effect.

The component then renders one small `MetricCard` per entry in `metrics` —
Deployment Frequency, Change Failure Rate, Lead Time, and Time to
Recovery. Two of those cards — the two proxies, Lead Time and Time to
Recovery — receive an `isProxy` prop that conditionally renders a small
"proxy" badge next to the number. This is Chapter 11's "always label a
proxy honestly, right next to the number" lesson, made concrete as
literal UI: nobody looking at this dashboard has to already know which of
the four numbers are exact and which are approximations standing in for
something unmeasurable — the badge tells them, right where they're
looking.

Two formatting helpers finish the picture, both mentioned only briefly
here since they're really about unit-clarity, a topic
[Chapter 27](27-polish-accessibility-and-bugfixes.md) returns to in full:
`formatRate` takes Deployment Frequency's raw "per day" number and chooses
whichever of "X / day," "X / week," or "X / month" reads most naturally at
that magnitude (exactly the presentation rule Chapter 11 described).
`formatDuration` does the equivalent job for the two proxy metrics'
millisecond values, picking between "X sec," "X min," or "Xh Ym" depending
on how large the duration is, so nobody has to mentally divide a raw
millisecond count to understand how long a deploy actually took.

## Checkpoint

- [ ] `lib/dora.ts` exists, with `median`, `average`, and four private
      `compute*` functions, and exports only `computeDoraMetrics`.
- [ ] `FAILURE_STATUSES` is defined once, at module scope, and used by both
      `computeChangeFailureRate` and `computeMttrProxy`.
- [ ] `computeMttrProxy` groups executions into a `Map<string, ExecutionRecord[]>`
      keyed by `pipelineIdentifier` before comparing any timestamps.
- [ ] `components/dora-metrics.tsx` calls `computeDoraMetrics(executions)`
      directly in the component body, with no `loading` state involved.
- [ ] You can explain, in your own words, why only `computeDoraMetrics` is
      exported from `lib/dora.ts`, and why a `Map` (not four separate
      arrays, or a plain object) is the right tool for grouping executions
      by pipeline.

**This generalizes to:** any dashboard's metrics module, on any subject,
benefits from this exact shape — one small, private, pure function per
metric; shared math (`average`, `median`, or anything else two metrics
both need) pulled out once rather than duplicated; a `Set` for any
recurring membership check; a `Map` for any recurring "group these records
by some key" need; and exactly one exported entry point that bundles every
metric into one object, so nothing outside the module ever needs to know
how many metrics exist or how each is individually computed. And on the
display side: call a pure, synchronous calculation directly in your
component body — reserve `loading`/`error` state for genuinely
asynchronous work, like the network fetch from the previous chapter.

**This is Piece #6 from the anatomy table** in
[Chapter 0](00-introduction.md) — the Metrics Module — now fully built:
raw `ExecutionRecord`s in, one clean `DoraMetrics` object out, rendered by
a component that trusts the calculation completely and only worries about
how to display it.

Next: [Chapter 23 — The PPv3 Comparison](23-ppv3-comparison.md)
