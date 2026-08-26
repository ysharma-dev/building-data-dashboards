# Chapter 20 — Building the Filter Bar

**Skill:** building a cascading filter UI for any hierarchical dataset —
where picking a value in one dropdown determines what options are even
valid in the next one — plus the "view everything, unfiltered" escape
hatch every good filter UI needs. This exact interaction pattern shows up
anywhere your data has a parent/child structure: countries and cities,
categories and products, teams and members.

## Why cascading, not four independent dropdowns

Recall the hierarchy from [Chapter 10](10-harness-and-cd-pipelines.md):
Organization → Project → Pipeline. If you offered four completely
independent dropdowns — pick any org, any project, any pipeline, with no
relationship between them — a user could select a project that doesn't
even belong to the org they picked, which is meaningless. Instead, each
dropdown's *options* should depend on what was picked in the dropdown
before it: pick an org, and the Project dropdown should only ever offer
projects that actually belong to that org.

This is our first genuinely stateful, interactive component — time to put
[Chapter 6](06-what-is-react.md)'s `useState`/`useEffect` to real use.

## The component's state

Create `components/filter-bar.tsx`. Start with the imports and the piece of
state each of the three selections needs, plus the *loaded lists* they're
chosen from:

```tsx
"use client";

import { useEffect, useState } from "react";
import type { HarnessOrg, HarnessPipeline, HarnessProject } from "@/lib/types";

export const ALL_PIPELINES = "__all__";

export type FilterSelection = {
  org: string | null;
  project: string | null;
  pipeline: string | null;
};

export function FilterBar({
  onChange,
}: {
  onChange: (selection: FilterSelection) => void;
}) {
  const [orgs, setOrgs] = useState<HarnessOrg[] | null>(null);

  const [org, setOrg] = useState<string | null>(null);
  const [project, setProject] = useState<string | null>(null);
  const [pipeline, setPipeline] = useState<string | null>(null);

  const [projects, setProjects] = useState<HarnessProject[]>([]);
  const [loadingProjects, setLoadingProjects] = useState(false);
  const [pipelines, setPipelines] = useState<HarnessPipeline[]>([]);
  const [loadingPipelines, setLoadingPipelines] = useState(false);

  const loadingOrgs = orgs === null;
```

A few design choices worth pausing on:

- **`"use client"`** — recall [Chapter 7](07-what-is-nextjs.md): this
  component uses `useState` and responds to clicks, so it must be a Client
  Component, not the App Router's server-rendered default.
- **`ALL_PIPELINES = "__all__"`** is a *sentinel value* — a specific,
  unlikely-to-collide string used as a stand-in for a special meaning
  ("all pipelines," rather than one specific pipeline's real identifier).
  We'll see exactly how this gets offered and handled below. Exporting it
  as a named constant (rather than writing the literal string `"__all__"`
  in multiple places) means every file that needs to check for this case
  imports the same constant — if the sentinel's exact value ever needed to
  change, there'd be exactly one place to update it.
- **`onChange`** is how this component tells whoever's using it ("I own
  the state, but the parent needs to know when it changes") — a callback
  prop, the same pattern from [Chapter 6](06-what-is-react.md).
- **`loadingOrgs = orgs === null`** is a small but genuinely useful trick:
  rather than a *separate* `isLoadingOrgs` boolean state variable that you'd
  have to remember to keep in sync, "loading" is *derived* directly from
  the data itself being `null` (meaning "hasn't arrived yet") versus an
  actual array (even an empty one, meaning "arrived, and there happen to be
  zero"). One source of truth, instead of two variables that could
  theoretically disagree with each other.

## Loading the top of the hierarchy on mount

Organizations don't depend on any other selection — they're the top of the
hierarchy — so they load once, when the component first appears:

```tsx
  useEffect(() => {
    let cancelled = false;
    fetch("/api/orgs")
      .then((r) => r.json())
      .then((data) => {
        if (!cancelled) setOrgs(sortByName(Array.isArray(data) ? data : []));
      });
    return () => {
      cancelled = true;
    };
  }, []);
```

The empty dependency array `[]` (from [Chapter 6](06-what-is-react.md))
means this effect runs exactly once, when the component first mounts. The
`cancelled` flag and the returned cleanup function guard against a subtle
timing bug: if this component were to disappear from the screen *before*
the fetch finishes (unlikely here, but a real risk in components that can
unmount quickly), calling `setOrgs` after that point would be pointless at
best and, in older React versions, could log a warning. Checking
`if (!cancelled)` before calling `setOrgs` means a fetch that resolves
after the component is already gone simply does nothing, rather than
trying to update state that no longer matters.

`sortByName` is a tiny local helper, worth defining once and reusing for
all three lists:

```tsx
function sortByName<T extends { name: string }>(items: T[]): T[] {
  return [...items].sort((a, b) => a.name.localeCompare(b.name));
}
```

## Cascading: each selection triggers the next fetch

Here's the actual cascade. When the org changes, we reset everything
*below* it (project and pipeline, and their loaded lists) and kick off a
fresh fetch for that org's projects:

```tsx
  function handleOrgChange(value: string | null) {
    if (!value) return;
    setOrg(value);
    setProject(null);
    setPipeline(null);
    setProjects([]);
    setPipelines([]);
    onChange({ org: value, project: null, pipeline: null });

    setLoadingProjects(true);
    fetch(`/api/projects?org=${encodeURIComponent(value)}`)
      .then((r) => r.json())
      .then((data) => setProjects(sortByName(Array.isArray(data) ? data : [])))
      .finally(() => setLoadingProjects(false));
  }
```

This is the heart of the whole "cascading" idea: **changing a higher-level
selection must reset every lower-level selection**, not just leave stale
choices sitting there. If it didn't reset `project` and `pipeline`, a user
could switch orgs and be left with a "selected" project that doesn't even
belong to the new org — exactly the meaningless state we set out to avoid.

`handleProjectChange` follows the identical shape, one level down (reset
only `pipeline`, since project is now the "top" of what's changing; fetch
that project's pipelines):

```tsx
  function handleProjectChange(value: string | null) {
    if (!value) return;
    setProject(value);
    setPipeline(null);
    setPipelines([]);
    onChange({ org, project: value, pipeline: null });

    if (!org) return;
    setLoadingPipelines(true);
    fetch(`/api/pipelines?org=${encodeURIComponent(org)}&project=${encodeURIComponent(value)}`)
      .then((r) => r.json())
      .then((data) => setPipelines(sortByName(Array.isArray(data) ? data : [])))
      .finally(() => setLoadingPipelines(false));
  }
```

And finally, picking a pipeline is the *bottom* of the cascade — nothing
left to reset, no further fetch to trigger, just report the final complete
selection upward:

```tsx
  function handlePipelineChange(value: string | null) {
    if (!value) return;
    setPipeline(value);
    onChange({ org, project, pipeline: value });
  }
```

## The "all pipelines" escape hatch

Now, the sentinel value from earlier. In the JSX where we render the
Pipeline dropdown's options, we offer one extra choice *before* the real
pipelines, only when there are any pipelines to compare against at all:

```tsx
<SelectContent>
  {pipelines.length > 0 && (
    <SelectItem value={ALL_PIPELINES}>All pipelines in this project</SelectItem>
  )}
  {pipelines.map((p) => (
    <SelectItem key={p.identifier} value={p.identifier}>
      {p.name}
    </SelectItem>
  ))}
</SelectContent>
```

Because `ALL_PIPELINES` is passed through `handlePipelineChange` exactly
like any real pipeline identifier would be, nothing *inside* this
component needs to treat it specially — it flows out through `onChange`
just like a real selection. It's the *consumer* of this component (built
in [Chapter 21](21-fetching-and-displaying-executions.md)) that checks
`pipeline === ALL_PIPELINES` and decides to call the "all pipelines in a
project" API route from [Chapter 19](19-remaining-api-routes.md) instead
of the single-pipeline one. This is a clean separation of concerns: the
filter bar's only job is to represent "what did the user pick," including
this special case — deciding *what to do* about that pick belongs
elsewhere.

## Disabling what doesn't make sense yet

Each dropdown should be unusable until its prerequisite is satisfied — you
can't meaningfully pick a project before an org, or a pipeline before a
project:

```tsx
<Select value={project ?? ""} onValueChange={handleProjectChange} disabled={!org || loadingProjects}>
```

`disabled={!org || loadingProjects}` reads naturally: disabled if there's
no org selected yet, *or* if we're actively waiting on that org's projects
to load. Each dropdown's placeholder text should say exactly why it's
unusable in each state ("Select org first" vs. "Loading…" vs. "Select
project") — a small detail, but one that removes any guessing about
*why* a control won't respond.

## Checkpoint

- [ ] `components/filter-bar.tsx` exists, exports `FilterBar`,
      `FilterSelection`, and `ALL_PIPELINES`.
- [ ] Selecting an org populates the Project dropdown with only that org's
      projects, and resets any previously-selected project/pipeline.
- [ ] Selecting a project populates the Pipeline dropdown, including an
      "All pipelines in this project" option at the top.
- [ ] Each dropdown is disabled with an explanatory placeholder until its
      prerequisite selection has been made.

**This generalizes to:** any dataset with a parent/child hierarchy needs
exactly this pattern — reset every downstream selection the moment an
upstream one changes, derive "loading" from the data itself rather than a
separate flag when you can, and offer a clear "view everything" escape
hatch using a sentinel value that flows through your normal state exactly
like a real selection would, leaving the decision about what that sentinel
*means* to whatever consumes the selection.

**This is Piece #5 from the anatomy table** in
[Chapter 0](00-introduction.md) — the Filter UI. In
[Chapter 29](29-building-your-own-dashboard.md), you'll design this exact
piece again for a completely different hierarchy.

Next: [Chapter 21 — Fetching and Displaying Executions](21-fetching-and-displaying-executions.md)
