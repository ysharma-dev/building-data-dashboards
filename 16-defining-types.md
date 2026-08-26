---
layout: chapter
title: "Chapter 16 — Defining Types"
nav_order: 17
permalink: /16-defining-types/
---

# Chapter 16 — Defining Types

**Skill:** modeling any external data source's entities as types *before*
writing code that uses them. This turns "I hope I remember every field this
object has and spell it right every time" into "the compiler tells me
immediately if I get it wrong." It's the single habit that makes every
later chapter in this book faster to write and safer to change.

## Why design types first

We haven't written a single line of Harness-calling code yet, and that's
deliberate. Before you write code that *produces* or *consumes* a piece of
data, it pays to stop and ask: **what shape does this data actually have?**
What fields does it have, what type is each field, and which fields might
be missing?

Answering that up front, as a `type` (see [Chapter 5](05-typescript-crash-course.md)
for a refresher), gives you two things immediately:
1. Every place you handle this data gets autocomplete and mistake-catching
   — if you typo a field name or forget one when constructing an object,
   TypeScript tells you *before* you run anything.
2. The type itself becomes documentation. A teammate (or future-you) can
   open `lib/types.ts` and understand your data model without reading every
   function that touches it.

Let's create that file now: `lib/types.ts`. First, make the `lib/` folder if
it doesn't exist yet, then create the file inside it.

## Modeling the hierarchy

Recall the Harness hierarchy from [Chapter 10](10-harness-and-cd-pipelines.md):
Organization → Project → Pipeline → Execution. Let's model the first three,
smallest ones first:

```ts
export type HarnessOrg = {
  identifier: string;
  name: string;
};

export type HarnessProject = {
  identifier: string;
  name: string;
  orgIdentifier: string;
};

export type HarnessPipeline = {
  identifier: string;
  name: string;
};
```

A few design decisions worth noticing:

- Every entity has both an `identifier` (a stable, machine-facing ID used
  in URLs and API calls — think of it like a username) and a `name` (a
  human-facing display label — think of it like a display name, which can
  contain spaces and can technically change without the identifier
  changing). This distinction — a stable key vs. a display label — shows up
  in almost every real-world data model you'll ever design.
- `HarnessProject` also carries `orgIdentifier` — a reference back to the
  org it belongs to. This is how you represent "belongs to a parent" in a
  flat type: not by nesting the whole parent object inside, just by
  carrying its ID.

## Modeling an execution

Now the more interesting entity: one single run of a pipeline. This is
where you'll spend most of the app's logic, so it's worth getting the shape
right.

```ts
export type StageDuration = {
  name: string;
  durationMs: number;
};

export type ChildStageLink = {
  stageName: string;
  childPlanExecutionId: string;
  childOrgId: string;
  childProjectId: string;
};

export type ExecutionRecord = {
  id: string;
  status: string;
  startTs: number;
  endTs: number;
  durationMs: number;
  stages: StageDuration[];
  childStageLinks: ChildStageLink[];
  pipelineIdentifier: string;
  pipelineName: string;
};
```

Walking through the design choices:

- **`startTs`/`endTs` are numbers, not `Date` objects.** They represent a
  moment in time as a *timestamp* — the number of milliseconds since a
  fixed reference point (January 1, 1970). Storing time as a plain number
  makes comparisons and arithmetic trivial (`endTs - startTs` is just
  subtraction) — you can always convert a timestamp into a human-readable
  date when you need to *display* it, which is a separate, later concern.
- **`durationMs` is stored, not computed on every use**, even though it
  could be derived every time as `endTs - startTs`. Storing a small,
  cheaply-recomputable value like this alongside its inputs is a reasonable
  convenience here — it saves callers from having to remember the
  subtraction every time they need a duration.
- **`stages: StageDuration[]`** — an execution isn't just one atomic block
  of time; it's made of multiple named steps, each with its own duration.
  Modeling this as an array of a small nested type (rather than, say, a
  dozen loose fields) lets any number of stages exist without changing the
  type.
- **`childStageLinks: ChildStageLink[]`** — this is the "parent calls
  child pipeline" pattern foreshadowed in [Chapter 10](10-harness-and-cd-pipelines.md)
  and explained fully in [Chapter 12](12-the-option1-optimization-story.md).
  An execution might have zero, one, or several stages that each call out
  to a separate child pipeline execution — again, naturally modeled as an
  array.
- **`pipelineIdentifier`/`pipelineName` live directly on the execution**,
  even though you might think "isn't that redundant, since we already know
  which pipeline we asked for?" It matters once you start fetching
  executions across *multiple* pipelines at once (as the "all pipelines in
  this project" feature does, starting in [Chapter 20](20-building-the-filter-bar.md))
  — at that point, each individual execution needs to carry its own
  pipeline identity so you can tell them apart after they're combined into
  one list.

## A note on `status: string`

You might expect `status` to be a stricter type — something like
`"Success" | "Failed" | "Aborted" | "Running"` (a *union type*, from
[Chapter 5](05-typescript-crash-course.md)) rather than a plain `string`.
That would be reasonable, and you're welcome to tighten it that way in your
own version. This project keeps it as a plain `string` because the value
comes directly from the Harness API's own response, and being loose here
avoids the type breaking if Harness's API ever returns a status value this
list didn't anticipate. Where a *stricter* set of known values genuinely
matters — as you'll see next — this project does use a proper union type.

## Modeling the Option 1 investigation

The most detailed type in the whole app supports the deep-dive investigation
from [Chapter 12](12-the-option1-optimization-story.md). Here, a stricter
union type earns its keep, because we genuinely only expect one of a small,
known set of values:

```ts
export type OptimizationStepStatus = "Skipped" | "Success" | "Failed" | "Unknown";

export type OptimizationOutcome = {
  stageName: string;
  parentExecutionId: string;
  childPlanExecutionId: string;
  manifestChanged: boolean | null;
  checkArgoStatus: OptimizationStepStatus;
  checkArgoDurationMs: number | null;
  deployApplicationStatus: OptimizationStepStatus;
  deployApplicationDurationMs: number | null;
  optimizationFired: boolean;
  approveDeployStatus: OptimizationStepStatus;
  approveDeployDurationMs: number | null;
  approveDeployStartTs: number | null;
  approveDeployEndTs: number | null;
  validateClustersStatus: OptimizationStepStatus;
  validateClustersDurationMs: number | null;
  validateClustersStartTs: number | null;
  validateClustersEndTs: number | null;
};
```

Notice the pervasive `| null` on nearly every field beyond the first three.
This is intentional, and it's a direct reflection of a real-world fact you
learned in Chapter 12: not every field is guaranteed to exist for every
execution. A step that was never reached has no duration to report; a value
that couldn't be determined is `null` rather than some made-up placeholder
number like `0` or `-1`. Writing `number | null` instead of just `number`
forces every piece of code that *reads* this field to explicitly handle
the "we don't know" case — TypeScript won't let you silently do arithmetic
on a value that might not exist. This is exactly the "isolate the confound,
don't quietly guess" principle from Chapter 12, now enforced by the type
system itself.

Also notice the four fields explicitly separating approval/validation wait
time (`approveDeploy*`, `validateClusters*`) from the deploy-mechanism
fields above them. This isn't an accident of naming — it's the type
directly encoding the confound-isolation lesson from Chapter 12: these
fields are *never* meant to be folded into `deployApplicationDurationMs` or
any skip-rate math. Keeping them as clearly, separately named fields makes
that intent visible to anyone reading the type, not just to whoever
remembers a comment somewhere else in the codebase.

## Modeling the computed metrics

Finally, a type for the *output* of the metrics calculations you'll build
in [Chapter 22](22-computing-metrics.md) — worth designing now, alongside
everything else, even though the calculation code doesn't exist yet:

```ts
export type DoraMetrics = {
  deploymentFrequency: {
    perDay: number;
    successCount: number;
    windowDays: number;
  };
  changeFailureRate: {
    ratePercent: number;
    failedCount: number;
    totalCount: number;
  };
  leadTimeProxyMs: {
    averageMs: number | null;
    medianMs: number | null;
    sampleCount: number;
  };
  mttrProxyMs: {
    averageMs: number | null;
    sampleCount: number;
  };
};
```

Notice each of the four metrics from [Chapter 11](11-metrics-explained.md)
returns not just a single headline number, but a small object that also
carries its own supporting detail (`successCount`, `sampleCount`, and so
on). This is a small but valuable design habit: a metric's raw number is
rarely useful entirely on its own — "2.3 per day" means something different
if it's built from 3 samples versus 300. Carrying the sample size alongside
the metric lets anything that *displays* this value show that context too,
which you'll see put to use directly in [Chapter 22](22-computing-metrics.md).

## Checkpoint

- [ ] `lib/types.ts` exists and exports `HarnessOrg`, `HarnessProject`,
      `HarnessPipeline`, `StageDuration`, `ChildStageLink`,
      `ExecutionRecord`, `OptimizationStepStatus`, `OptimizationOutcome`,
      and `DoraMetrics`.
- [ ] Running `npx tsc --noEmit` in your project (checks types across the
      whole project without producing any output files) completes with no
      errors.
- [ ] You can explain, in your own words, why several fields on
      `OptimizationOutcome` are typed as `number | null` instead of just
      `number`.

**This generalizes to:** whatever data source you build a dashboard around
next — a GitHub API, a spreadsheet export, a database table — start by
designing types for its entities exactly this way, before writing any
fetch or calculation code: identify the stable ID vs. the display name,
model one-to-many relationships as arrays, use `| null` honestly wherever a
value can genuinely be missing, and design the *output* shape of any
calculation you know you'll need, even before you've written that
calculation. The types become the skeleton everything else hangs on.

Next: [Chapter 17 — Talking to the Harness API](17-talking-to-the-harness-api.md)
