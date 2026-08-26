---
layout: chapter
title: "Chapter 21 — Fetching and Displaying Executions"
nav_order: 22
permalink: /21-fetching-and-displaying-executions/
---

# Chapter 21 — Fetching and Displaying Executions

**Skill:** the loading/error/data three-state pattern that every piece of
async UI needs — a piece of data you fetch is always, at any given moment,
in exactly one of three states: still loading, failed with an error, or
successfully loaded. Your UI has to render something sensible for *all
three*, not just the happy path where the data shows up instantly and
nothing ever goes wrong. Alongside that, this chapter builds a paginated
table — the standard way to show a long list of records without forcing
someone to scroll through hundreds of rows to find what they're looking
for.

## Why "just show the data" isn't enough

If you've only ever written toy examples, it's tempting to write a
component that assumes the data is already there: `executions.map(...)`,
done. But think about what actually happens the instant a user picks a
pipeline in the filter bar you built in [Chapter 20](20-building-the-filter-bar.md):
there's a real network request in flight, and for some number of
milliseconds — sometimes a very noticeable number, if Harness's API is
slow or the connection is bad — there is genuinely no data to show yet.
And sometimes the request fails outright: the API key expired, the network
dropped, Harness returned a 500. If your component only knows how to
render "the data," it has nothing sensible to draw in either of those two
very real situations.

So every fetch you write, for the rest of your career, should be modeled
as three explicit states, all tracked side by side. Let's see this built
for real.

## The state, in `app/page.tsx`

`app/page.tsx` is the top of this app — the page component that owns the
current filter selection and the executions fetched for it. Here's the
state it declares:

```tsx
"use client";

import { useMemo, useRef, useState } from "react";
import { ALL_PIPELINES, FilterBar, type FilterSelection } from "@/components/filter-bar";
import { StatusFilter, type StatusOption } from "@/components/status-filter";
import type { ExecutionRecord } from "@/lib/types";

type ProjectExecutionsResponse = {
  pipelineIdentifier: string;
  pipelineName: string;
  executions: ExecutionRecord[];
};

export default function Home() {
  const [selection, setSelection] = useState<FilterSelection>({
    org: null,
    project: null,
    pipeline: null,
  });
  const [executions, setExecutions] = useState<ExecutionRecord[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [status, setStatus] = useState<StatusOption>("All");
```

Notice there are *three separate pieces of state* directly mirroring the
three-state idea: `executions` (the data, starting as an empty array),
`loading` (a boolean, starting `false` since nothing is being fetched
yet), and `error` (starting `null`, meaning "no error has happened").
`ProjectExecutionsResponse` is a small local type describing exactly what
shape comes back from the "all pipelines in a project" API route you'll
see below — one entry per pipeline, each carrying that pipeline's own
executions.

## Handling a new selection

Every time the filter bar reports a new complete selection, this function
runs:

```tsx
  function handleSelectionChange(next: FilterSelection) {
    setSelection(next);

    const { org, project, pipeline } = next;
    if (!org || !project || !pipeline) {
      setExecutions([]);
      setError(null);
      return;
    }

    setLoading(true);
    setError(null);

    const fetchPromise: Promise<ExecutionRecord[]> =
      pipeline === ALL_PIPELINES
        ? fetch(
            `/api/executions/project?${new URLSearchParams({ org, project }).toString()}`,
          ).then(async (r) => {
            const data = await r.json();
            if (!r.ok) throw new Error(data.error ?? "Failed to load executions");
            return (data as ProjectExecutionsResponse[]).flatMap((d) => d.executions);
          })
        : fetch(
            `/api/executions?${new URLSearchParams({ org, project, pipeline, limit: "100" }).toString()}`,
          ).then(async (r) => {
            const data = await r.json();
            if (!r.ok) throw new Error(data.error ?? "Failed to load executions");
            return data as ExecutionRecord[];
          });

    fetchPromise
      .then((data) => setExecutions(Array.isArray(data) ? data : []))
      .catch((err) => setError(err.message))
      .finally(() => setLoading(false));
  }
```

Let's walk through this piece by piece.

**The incomplete-selection guard.** `if (!org || !project || !pipeline)`
handles the moment right after the app first loads, or right after someone
changes the org (which, recall from Chapter 20, resets project and
pipeline back to `null`). With no complete selection, there's nothing to
fetch — so this branch just clears out any previous executions and error,
and returns immediately without ever touching `loading`.

**Two different requests, chosen by which pipeline was picked.** This is
the interesting branch. Recall from Chapter 20 that `ALL_PIPELINES` is a
sentinel value — a stand-in for "don't fetch one pipeline, fetch every
pipeline in this project." That decision is made right here, at the
*consumer* of the filter bar, exactly as Chapter 20 promised:

- If `pipeline === ALL_PIPELINES`, we call `/api/executions/project` — an
  API route (built the way you learned in [Chapter 18](18-first-api-route.md)
  and [Chapter 19](19-remaining-api-routes.md)) that internally uses
  `listExecutionsForProject` from [Chapter 17](17-talking-to-the-harness-api.md)
  to fetch every pipeline's executions concurrently. It responds with an
  array of `ProjectExecutionsResponse` objects — one per pipeline, each
  bundling that pipeline's own list of executions.
- Otherwise, we call `/api/executions` for that one specific pipeline,
  which responds with a flat `ExecutionRecord[]` directly.

Both branches end up needing to produce the *same* final shape —
`Promise<ExecutionRecord[]>` — so that everything after this `if/else`
(the `.then().catch().finally()` chain) can treat them identically without
caring which branch ran. That's why the "all pipelines" branch ends with
`.flatMap((d) => d.executions)`. `flatMap` is a single operation that does
what `.map()` followed by `.flat()` would do in two steps: it runs a
function over every item (here, "pull out this pipeline's `executions`
array") and then flattens all those individual arrays into one combined
array, instead of leaving you with an array *of* arrays. If pipeline A
returned 12 executions and pipeline B returned 8, `.flatMap()` here gives
you one flat array of 20 — exactly the shape the single-pipeline branch
produces directly.

**Checking `r.ok` before trusting the body.** Both branches call
`await r.json()` first, then check `if (!r.ok) throw new Error(...)`. This
mirrors the same "fail loud with a useful message" habit from
`harnessFetch` in Chapter 17 — an API route can respond with a non-2xx
status *and* a JSON body describing what went wrong (`{ error: "..." }`),
so we read the body regardless, and only decide whether to treat it as
success or failure afterward.

**The final chain.** `.then((data) => setExecutions(...))` stores
whichever promise resolved with, guarding once more with
`Array.isArray(data)` in case something unexpected came back.
`.catch((err) => setError(err.message))` catches *either* branch's
failure — a network failure, a thrown `Error` from the `!r.ok` check, or
anything else — and stores its message. `.finally(() => setLoading(false))`
always runs last, whether the fetch succeeded or failed, guaranteeing you
never get stuck showing a loading state forever. This is the general
shape of the pattern: set `loading` true and `error` null right before you
start, then let `.then` handle success, `.catch` handle failure, and
`.finally` turn `loading` back off unconditionally.

## `Boolean(...)`, and a bug we'll return to

Just below this, the component computes:

```tsx
  const hasSelection = Boolean(selection.org && selection.project && selection.pipeline);
```

You might wonder why this doesn't just write
`const hasSelection = selection.org && selection.project && selection.pipeline`
and use *that* directly. The difference matters more than it looks: without
`Boolean(...)`, that expression doesn't evaluate to `true` or `false` at
all — thanks to how JavaScript's `&&` operator works, it evaluates to
whichever operand it stopped on, which could be `null` or an actual string
like `"myorg"`, not a real boolean. Wrapping the whole thing in `Boolean(...)`
forces it into an actual `true`/`false` value. This project actually hit a
real bug from skipping this step — a **hydration mismatch**, where the
page Next.js renders on the server disagrees with what React renders in
the browser. We'll dig into exactly why that happens and how it was fixed,
in full, in [Chapter 27](27-polish-accessibility-and-bugfixes.md). For now,
just take away the habit: when you need a true boolean for a rendering
decision, make it an actual boolean explicitly, rather than trusting
whatever a chain of `&&` happens to evaluate to.

## The three-state render

Here's the part that ties this whole chapter together. Conceptually (you'll
type the exact JSX yourself, but the shape is what matters), the page's
render logic checks four conditions, in a fixed order, and renders exactly
one of four things:

1. **`!hasSelection`** — nothing has been fully picked yet. Show a calm
   placeholder message, something like "Pick an org, project, and pipeline
   to see deployment metrics." No spinner, no error — there's nothing wrong,
   the user just hasn't finished choosing yet.
2. **`error`** — the fetch failed. Show a clearly-styled error box
   containing the message you stored in `setError`, so whoever's looking at
   the screen knows *something specific* went wrong, not just "it's blank."
3. **`loading`** — a fetch is currently in flight. Show skeleton
   placeholders: gray, pulsing rectangles roughly the shape of the content
   that's about to appear, using shadcn's `Skeleton` component (from
   [Chapter 9](09-what-is-shadcn-ui.md)). A skeleton is a small but real
   trust-building detail — it tells the user "this exact area is about to
   fill in," rather than leaving a jarring blank gap or a generic spinner
   disconnected from the actual layout that's coming.
4. **Otherwise** — none of the above apply, which means: a selection exists,
   there's no error, and nothing is loading. Only now do we render the real
   content — the metrics cards, the comparison view, and the table you're
   about to build below.

This is the general "loading / error / data" pattern named at the top of
this chapter, made concrete. The important detail isn't just that all four
cases exist — it's that they're checked **in a fixed order, each one
exclusive of the ones before it** (an `if / else if / else if / else`
chain, or the equivalent using early returns). Checking "not selected"
first, then "error," then "loading," and falling through to "success" only
at the very end guarantees that exactly one of the four ever renders at
once. If you instead wrote four independent, unordered `if` blocks (or
worse, four sibling `{condition && <Thing/>}` expressions with no
relationship between them), you risk a moment where two conditions are
accidentally both true — say, a stale `error` from a previous failed fetch
still sitting in state *while* a new fetch is `loading` — and you'd render
an error box and a skeleton stacked on top of each other. Ordering the
checks, and treating them as mutually exclusive branches of one decision
rather than four independent ones, is what prevents that.

## Filtering without re-filtering on every render

One more piece of state-derived logic sits right after this:

```tsx
  const filteredExecutions = useMemo(() => {
    if (status === "All") return executions;
    return executions.filter((e) => e.status === status);
  }, [executions, status]);
```

`status` here comes from the `StatusFilter` component — a simpler sibling
of the filter bar that lets someone narrow the currently-loaded executions
down to just one status ("Success," "Failed," and so on) without making a
new network request; it just filters what's already sitting in memory.

Why wrap this in `useMemo` (from [Chapter 6](06-what-is-react.md)) instead
of just writing `const filteredExecutions = status === "All" ? executions : executions.filter(...)`
directly in the component body? Every time this component re-renders — and
components re-render for all kinds of reasons that have nothing to do with
`executions` or `status` changing, like a parent re-rendering, or unrelated
state changing elsewhere on the page — that plain version would re-run the
`.filter()` call from scratch, even when the result would be identical to
last time. `useMemo(fn, [executions, status])` tells React: "only re-run
this calculation when `executions` or `status` has actually changed since
last time; on any other re-render, just hand back the previously computed
array." For a few dozen executions this wouldn't be noticeable either way
— but the *habit* matters: `useMemo` is the right tool anywhere you derive
one piece of data from others via a nontrivial calculation, and reaching
for it consistently means you never have to stop and wonder "is this
recalculating more than it needs to?" later, once the list is a lot bigger
or the calculation is a lot heavier (as it will be starting in
[Chapter 22](22-computing-metrics.md)).

## Building a paginated table

With `filteredExecutions` in hand, we need somewhere to actually show every
individual execution — not just the summary metrics, but the raw list,
browsable row by row. A pipeline that's been running for months could
easily have hundreds of executions; rendering all of them into one giant
`<table>` would make the page slow to render and painful to scroll through.
The standard fix is **pagination**: show a fixed-size "page" of rows at a
time, with controls to move to the next or previous page.

Create `components/execution-table.tsx`:

```tsx
"use client";

import { Fragment, useMemo, useState } from "react";
import type { ExecutionRecord } from "@/lib/types";

const PAGE_SIZE = 25;

export function ExecutionTable({
  executions,
  cutoffTs,
}: {
  executions: ExecutionRecord[];
  cutoffTs: number | null;
}) {
  const [expanded, setExpanded] = useState<Set<string>>(new Set());
  const [page, setPage] = useState(0);

  function toggle(id: string) {
    setExpanded((prev) => {
      const next = new Set(prev);
      if (next.has(id)) next.delete(id);
      else next.add(id);
      return next;
    });
  }
```

`PAGE_SIZE = 25` is a module-level constant — a fixed number of rows shown
per page, chosen once and referred to everywhere below by name rather than
as a "magic number" repeated in multiple places. `cutoffTs` is a prop
you'll put to use starting in [Chapter 24](24-the-option1-deep-dive.md); for
this chapter, just know it flows in from a parent component and isn't part
of the pagination logic itself.

**Why a `Set<string>` for `expanded`, not an array.** Each row in this
table is clickable, and clicking it should expand that one row to show a
detailed per-stage duration breakdown underneath it (using the `stages`
array on `ExecutionRecord`, from [Chapter 16](16-defining-types.md)). The
question the UI needs to answer, over and over, as it renders each row, is:
"is *this specific execution's ID* currently expanded?" A `Set` is built
exactly for that question — checking whether a value is a member
(`.has(id)`) is fast and direct, regardless of how many IDs are in the
set. An array would force you to write `expandedArray.includes(id)`,
which has to scan through the array checking each entry one by one. For a
small table this difference is invisible, but the *shape* of the tool
matters more than the speed here: "is X a member of this collection" is
precisely what a `Set` is for, and reaching for one signals that intent
clearly to anyone reading the code.

**The immutable-update pattern inside `toggle`.** Look closely at what
`toggle` does: `const next = new Set(prev)` makes a brand-new `Set`,
copying every element `prev` had, and *then* calls `.delete()` or `.add()`
on that new copy — never on `prev` itself. This matters for a very
specific reason: React decides whether to re-render a component by
comparing a piece of state's *previous reference* to its *new reference*
— not by deeply inspecting whether the contents changed. If `toggle`
instead did `prev.delete(id); setExpanded(prev);` — mutating the existing
`Set` in place and then handing that *same* object back to `setExpanded`
— React would see the exact same reference it already had, conclude
"nothing changed," and skip the re-render entirely, even though the set's
contents are now different. Copying first (`new Set(prev)`), then
mutating only the copy, guarantees `setExpanded` receives a genuinely new
object reference every time, so React reliably notices the change and
re-renders. This "copy, then mutate the copy" habit is not specific to
`Set` — you'll want the same discipline for arrays (`[...prev]`) and plain
objects (`{ ...prev }`) any time you update React state that isn't a
plain primitive.

## Pagination math

```tsx
  const sorted = useMemo(
    () => [...executions].sort((a, b) => b.startTs - a.startTs),
    [executions],
  );

  // Clamping (rather than resetting via an effect) keeps this correct even
  // when `executions` changes: a shrunk list can never leave `page` pointing
  // past the end, without needing a render-triggering effect to catch up.
  const pageCount = Math.max(Math.ceil(sorted.length / PAGE_SIZE), 1);
  const clampedPage = Math.min(page, pageCount - 1);
  const pageStart = clampedPage * PAGE_SIZE;
  const pageRows = sorted.slice(pageStart, pageStart + PAGE_SIZE);
```

`sorted` copies `executions` (`[...executions]`, so we never mutate the
original array the parent handed us) and sorts newest-first by `startTs`,
memoized so this only re-sorts when `executions` itself changes — the same
`useMemo` habit from earlier in this chapter.

Then the pagination math, four small steps:

- **`pageCount`** — how many pages of `PAGE_SIZE` rows it takes to cover
  every row in `sorted`. `Math.ceil(sorted.length / PAGE_SIZE)` rounds up,
  since a partially-full last page (say, 7 leftover rows) still counts as
  one whole page. `Math.max(..., 1)` guarantees at least one page even when
  `sorted` is empty — you always want a well-defined "page 1 of 1," never
  "page 1 of 0."
- **`clampedPage`** — the page you're actually going to show, guaranteed to
  never exceed the last valid page index (`pageCount - 1`). `Math.min(page, pageCount - 1)`
  reads as "whichever is smaller: the page the user last navigated to, or
  the last page that actually exists."
- **`pageStart`** — the index, within `sorted`, where this page's rows
  begin: page 0 starts at row 0, page 1 starts at row 25, and so on.
- **`pageRows`** — the actual slice of rows to render this time, taken with
  the built-in `.slice(start, end)` array method, which returns everything
  from `pageStart` up to (but not including) `pageStart + PAGE_SIZE`.

The rows below render `pageRows` — each one clickable via `toggle(id)`, and
each one, when its ID is present in `expanded`, showing an extra row
underneath with a per-stage duration breakdown built from that execution's
`stages` array.

## Why clamp during render, not reset in an effect

The comment right above the pagination math calls out a specific design
decision worth understanding on its own: **`clampedPage` is computed
directly, every render, from `page` and `pageCount` — there is no
`useEffect` watching `executions.length` and calling `setPage(0)` when it
shrinks.**

It's worth being explicit about the bug this avoids. Imagine you're on page
3 of a table, and then a filter change (like the status filter from
earlier in this chapter) shrinks `executions` down to only one page's
worth of rows. If `page` itself stayed at `3` and nothing corrected it,
`pageStart` would be computed way past the end of the array, and
`.slice()` would silently return an empty array — the table would just go
blank, with no indication of what happened.

The tempting fix is a `useEffect` that watches for this and calls
`setPage(0)` to reset it. But that approach means: render with the stale,
out-of-range `page`, notice in an effect that it's now invalid, call
`setPage(0)`, and then render *again* with the corrected value — two
renders to reach a correct result, and a brief flash of the broken
in-between state on the first one. Computing `clampedPage` directly during
render sidesteps all of that: it's a pure calculation from values you
already have (`page` and `pageCount`), so the very first render already
shows the correct, in-range page — no effect, no extra render, no
flash of wrong state.

This isn't just a style preference in this project — its ESLint
configuration (recall linters from [Chapter 2](02-installing-tools.md))
specifically flags "calling `setState` inside a `useEffect` to keep one
piece of state in sync with another" as usually avoidable, precisely
because it so often can be replaced by deriving the corrected value
directly during render, the way `clampedPage` does here. Whenever you
catch yourself reaching for an effect whose entire job is "recompute this
other state whenever that state changes," pause and ask whether you can
just compute the corrected value inline instead, the way this table does.

## Checkpoint

- [ ] `app/page.tsx` tracks `executions`, `loading`, and `error` as
      separate state, and `handleSelectionChange` chooses between
      `/api/executions` and `/api/executions/project` based on whether
      `pipeline === ALL_PIPELINES`.
- [ ] The page renders exactly one of: a "make a selection" placeholder, an
      error box, loading skeletons, or the real content — checked in that
      order.
- [ ] `components/execution-table.tsx` exists, paginates `executions` at
      25 rows per page, and lets each row expand to show its stage
      breakdown.
- [ ] You can explain, in your own words, why `toggle` builds a `new Set(prev)`
      instead of mutating `prev` directly, and why `clampedPage` is computed
      during render instead of reset via a `useEffect`.

**This generalizes to:** any async UI, on any subject, needs the same
three states tracked explicitly — loading, error, and data — checked in a
fixed, mutually-exclusive order so you never accidentally render two of
them at once. And any list of records long enough to matter — orders,
log lines, support tickets, running totals — benefits from the same
paginate-by-slicing approach: sort once, compute how many fixed-size pages
that implies, clamp the current page directly during render rather than
correcting it after the fact in an effect, and slice out just the rows
that page needs.

**This is Piece #3 from the anatomy table** in [Chapter 0](00-introduction.md)
— the Fetch Layer — put to use for the first time on-screen: this chapter
is where the functions built in Chapter 17 and the routes built in
Chapters 18-19 finally reach a real, interactive UI, and the loading/error/
data pattern you just built will reappear, unchanged in spirit, under every
single view the rest of this book adds.

Next: [Chapter 22 — Computing Metrics](22-computing-metrics.md)
