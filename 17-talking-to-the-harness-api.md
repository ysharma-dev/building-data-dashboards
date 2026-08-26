---
layout: chapter
title: "Chapter 17 — Talking to the Harness API"
nav_order: 18
permalink: /17-talking-to-the-harness-api/
---

# Chapter 17 — Talking to the Harness API

**Skill:** writing a data-access layer — a small, dedicated module that is
the *only* place in your entire app that knows how one particular external
API actually works. Every other file in the app calls simple, well-named
functions like `listOrgs()` and never has to know about URLs, headers, or
response shapes. This is one of the most valuable habits in this whole
book: it means when an API changes, or you swap it out entirely, you have
exactly one file to update.

## Why isolate all API knowledge in one file

Imagine you *didn't* do this — every component that needed a list of
organizations just called `fetch()` directly, wherever it needed the data.
Now imagine Harness changes its API's URL structure, or you decide to
switch to a completely different data source. You'd have to hunt down every
single place that called `fetch()` and update each one, hoping you don't
miss any.

Instead, we're going to build one file — `lib/harness.ts` — that exports a
small number of clearly-named async functions (`listOrgs`, `listProjects`,
`listPipelines`, `listExecutions`, and so on). Everything else in the app
calls *those* functions and never touches a URL or a header directly. If
Harness's API changes, exactly one file needs to change.

## The shared request helper

Every single call to Harness needs the same three things: the base URL, an
API key in a header, and consistent error handling if something goes
wrong. Rather than repeating that in every function, we write it once:

```ts
function getConfig() {
  const apiKey = process.env.HARNESS_API_KEY;
  const accountId = process.env.HARNESS_ACCOUNT_ID;
  const baseUrl = process.env.HARNESS_BASE_URL;

  if (!apiKey || !accountId || !baseUrl) {
    throw new Error(
      "Missing Harness config. Set HARNESS_API_KEY, HARNESS_ACCOUNT_ID, HARNESS_BASE_URL in .env.local",
    );
  }

  return { apiKey, accountId, baseUrl };
}

async function harnessFetch(path: string, init?: RequestInit) {
  const { apiKey, baseUrl } = getConfig();

  const res = await fetch(`${baseUrl}${path}`, {
    ...init,
    headers: {
      "x-api-key": apiKey,
      "Content-Type": "application/json",
      ...init?.headers,
    },
    cache: "no-store",
  });

  if (!res.ok) {
    const body = await res.text().catch(() => "");
    throw new Error(`Harness API ${res.status} ${res.statusText}: ${body.slice(0, 300)}`);
  }

  return res.json();
}
```

You saw `getConfig()` already in [Chapter 15](15-environment-variables-and-secrets.md)
— it's the same loud-failure-on-missing-config function.

`harnessFetch` is the one place that actually calls the browser/Node.js
built-in `fetch()` function to make an HTTP request. A few details worth
understanding:

- `${baseUrl}${path}` builds the full URL by joining the configured base
  (e.g. `https://app.harness.io`) with whatever specific path each calling
  function provides (e.g. `/ng/api/organizations`).
- The `x-api-key` header is how Harness's API authenticates the request —
  this is the secret from your `.env.local` actually being used.
- `...init?.headers` at the end means any headers a caller passes in via
  `init` can add to (or override) these defaults — the `...` spread syntax
  you learned in [Chapter 4](04-javascript-crash-course.md).
- `cache: "no-store"` tells Next.js's built-in `fetch` caching (a feature
  that can otherwise reuse a previous response instead of making a fresh
  request) to never do that here — dashboard data needs to be current every
  time, not stale from an earlier request.
- `if (!res.ok)` checks whether the HTTP response indicates success. If
  not, we read the response body as text (wrapped in `.catch(() => "")` in
  case even *that* fails) and throw a descriptive error — again, the
  "fail loud with useful detail" pattern from Chapter 15.

## The mock-data check

Recall the `USE_MOCK_DATA` flag from [Chapter 15](15-environment-variables-and-secrets.md).
Every public function below starts with the same one-line check:

```ts
function useMockData(): boolean {
  return process.env.USE_MOCK_DATA === "true";
}
```

You'll see this called at the top of each function that follows. The full
mock data itself (`mockOrgs()`, `mockProjects()`, and so on) is built out
in [Appendix A](appendix-a-mock-data-fallback.md) — for this chapter, focus
on the real-API code path; just know that one `if` at the top of each
function is what makes the whole app work without a Harness account.

## Listing organizations and projects

With the shared helper in place, the actual entity-fetching functions
become short and readable:

```ts
function byName<T extends { name: string }>(a: T, b: T): number {
  return a.name.localeCompare(b.name);
}

export async function listOrgs(): Promise<HarnessOrg[]> {
  if (useMockData()) return mockOrgs();

  const { accountId } = getConfig();
  const json = await harnessFetch(
    `/ng/api/organizations?accountIdentifier=${accountId}&pageSize=200`,
  );
  const content = json.data?.content ?? [];
  return content
    .map((c: { organization: { identifier: string; name: string } }) => ({
      identifier: c.organization.identifier,
      name: c.organization.name,
    }))
    .sort(byName);
}
```

Two small but important habits here:

- **`byName`** is a tiny, reusable comparator function (the kind
  `.sort()` expects, from [Chapter 4](04-javascript-crash-course.md)) that
  sorts anything with a `name` field alphabetically, case-sensitively-aware
  via `.localeCompare()` (a string comparison method that handles
  alphabetical ordering correctly across languages/locales, better than a
  plain `<` comparison). Writing this once, generically, means every
  entity list in the app can be sorted the same consistent way.
- **`json.data?.content ?? []`** — this is doing real defensive work.
  Harness's raw response wraps the actual list two levels deep
  (`data.content`), and the `?.` (optional chaining, from
  [Chapter 5](05-typescript-crash-course.md)) means "if `json.data` doesn't
  exist, don't crash — just produce `undefined`," and the `?? []`
  (nullish coalescing) means "if that came back `undefined`, use an empty
  array instead." The result: if Harness's response is ever shaped
  unexpectedly, this function returns an empty list rather than crashing
  the whole app.
- **The `.map(...)` step reshapes Harness's raw response into *our own*
  type** (`HarnessOrg`, from [Chapter 16](16-defining-types.md)) — picking
  out only the two fields we actually care about, `identifier` and `name`,
  and discarding whatever else Harness's response happens to include. This
  is the isolation this whole chapter is about: nothing outside this file
  ever sees Harness's raw response shape, only our own clean type.

`listProjects` follows the identical shape, one level deeper in the
hierarchy — it takes an `org` identifier as a parameter and includes it in
the query string:

```ts
export async function listProjects(org: string): Promise<HarnessProject[]> {
  if (useMockData()) return mockProjects(org);

  const { accountId } = getConfig();
  const json = await harnessFetch(
    `/ng/api/projects?accountIdentifier=${accountId}&orgIdentifier=${org}&pageSize=500`,
  );
  const content = json.data?.content ?? [];
  return content
    .map(
      (c: { project: { identifier: string; name: string; orgIdentifier: string } }) => ({
        identifier: c.project.identifier,
        name: c.project.name,
        orgIdentifier: c.project.orgIdentifier,
      }),
    )
    .sort(byName);
}
```

## Listing pipelines, and a simple classifier

Pipelines follow the same pattern, but the underlying Harness endpoint
happens to require a `POST` request with a small JSON body rather than a
plain `GET`:

```ts
export async function listPipelines(org: string, project: string): Promise<HarnessPipeline[]> {
  if (useMockData()) return mockPipelines(org, project);

  const { accountId } = getConfig();
  const json = await harnessFetch(
    `/pipeline/api/pipelines/list?accountIdentifier=${accountId}&orgIdentifier=${org}&projectIdentifier=${project}&page=0&size=200`,
    {
      method: "POST",
      body: JSON.stringify({ filterType: "PipelineSetup" }),
    },
  );
  const content = json.data?.content ?? [];
  return content
    .map((c: { identifier: string; name: string }) => ({
      identifier: c.identifier,
      name: c.name,
    }))
    .sort(byName);
}
```

Alongside pipeline-listing, here's a genuinely simple function, worth
including in this same file since it's about classifying a `HarnessPipeline`
— the "PPv3" naming-convention classifier from
[Chapter 12](12-the-option1-optimization-story.md):

```ts
const PPV3_PATTERN = /ppv3/i;

export function isPpv3Pipeline(pipeline: HarnessPipeline): boolean {
  return PPV3_PATTERN.test(pipeline.identifier) || PPV3_PATTERN.test(pipeline.name);
}
```

`/ppv3/i` is a **regular expression** (often shortened to "regex") — a
compact pattern language for matching text. This one matches the literal
text `ppv3` anywhere in a string, and the trailing `i` flag makes it
case-insensitive, so it matches `PPv3`, `ppv3`, `PPV3`, and so on equally.
`.test(someString)` returns `true` or `false` depending on whether the
pattern matches. This function is *not* async and doesn't call the network
at all — it's a pure classification function that works on data you
already have in memory, which is why it doesn't need the `useMockData()`
check the fetching functions above do.

## Turning a raw execution into our own shape

Here's where things get more involved: turning one raw Harness execution
response into our clean `ExecutionRecord` type from
[Chapter 16](16-defining-types.md). Harness represents an execution's
internal steps as a `layoutNodeMap` — an object where each key is some
internal node ID and each value describes one node (a stage, a step, and
so on) in that execution's graph:

```ts
function normalizeExecution(
  e: RawExecution,
  pipelineIdentifier: string,
  pipelineName: string,
): ExecutionRecord {
  const nodes = Object.values(e.layoutNodeMap ?? {});

  const stages = nodes
    .filter((n) => n.name && n.startTs && n.endTs)
    .map((n) => ({
      name: n.name as string,
      durationMs: (n.endTs as number) - (n.startTs as number),
    }));

  const childStageLinks = nodes
    .filter(
      (n) =>
        n.nodeType === "Pipeline" &&
        !!n.name &&
        !!n.stepDetails?.childPipelineExecutionDetails?.planExecutionId,
    )
    .map((n) => ({
      stageName: n.name,
      childPlanExecutionId: n.stepDetails.childPipelineExecutionDetails.planExecutionId,
      childOrgId: n.stepDetails.childPipelineExecutionDetails.orgId,
      childProjectId: n.stepDetails.childPipelineExecutionDetails.projectId,
    }));

  return {
    id: e.planExecutionId,
    status: e.status,
    startTs: e.startTs,
    endTs: e.endTs,
    durationMs: e.endTs - e.startTs,
    stages,
    childStageLinks,
    pipelineIdentifier,
    pipelineName,
  };
}
```

`Object.values(e.layoutNodeMap ?? {})` turns that node-ID-keyed object into
a plain array we can `.filter()` and `.map()` over — the object's *keys*
(the internal node IDs) don't matter to us, only the node *values*
themselves.

The two `.filter().map()` chains are each doing exactly the same kind of
work you practiced in [Chapter 4](04-javascript-crash-course.md): keep only
the nodes that look like what we want, then reshape each surviving node
into our own clean type.

- **`stages`** keeps every node that has a `name`, a `startTs`, and an
  `endTs` — i.e., every node that represents *some* timed step, regardless
  of what kind — and turns each into a `{ name, durationMs }` pair.
- **`childStageLinks`** is far more specific: it keeps only nodes where
  `nodeType === "Pipeline"` (meaning this stage calls a separate child
  pipeline — exactly the pattern from Chapter 12) *and* that node actually
  has a `childPipelineExecutionDetails.planExecutionId` present (the ID we'd
  need to go fetch that child execution). Most executions will have zero
  nodes matching this — that's expected; only pipelines built around the
  parent-calls-child pattern will have any.

## Fetching a list of executions, with pagination

Harness's execution-listing endpoint returns results a page at a time
(a common pattern for any API returning potentially large lists — never
hand you 10,000 records in one response). `listExecutions` handles that by
looping, requesting one page after another until it has enough results or
runs out of pages:

```ts
export async function listExecutions(
  org: string,
  project: string,
  pipeline: string,
  limit = 100,
  pipelineName: string = pipeline,
): Promise<ExecutionRecord[]> {
  if (useMockData()) return mockExecutions(pipeline, pipelineName, limit);

  const { accountId } = getConfig();
  const pageSize = Math.min(limit, 100);
  const results: ExecutionRecord[] = [];
  let page = 0;

  while (results.length < limit) {
    const json = await harnessFetch(
      `/pipeline/api/pipelines/execution/summary?accountIdentifier=${accountId}&orgIdentifier=${org}&projectIdentifier=${project}&pipelineIdentifier=${pipeline}&page=${page}&size=${pageSize}`,
      {
        method: "POST",
        body: JSON.stringify({ filterType: "PipelineExecution" }),
      },
    );

    const content: RawExecution[] = json.data?.content ?? [];
    if (content.length === 0) break;

    results.push(...content.map((e) => normalizeExecution(e, pipeline, pipelineName)));

    const totalPages = json.data?.totalPages ?? 1;
    page += 1;
    if (page >= totalPages) break;
  }

  return results
    .filter((e) => e.endTs && e.startTs)
    .sort((a, b) => a.startTs - b.startTs)
    .slice(0, limit);
}
```

Walking through the loop: it keeps requesting successive pages
(`page = 0`, then `1`, then `2`, ...) and appending each page's results to
`results`, until either (a) it has collected `limit` or more results, (b) a
page comes back completely empty (`content.length === 0`, meaning we've run
past the actual data), or (c) we've reached the last page the server told
us about (`page >= totalPages`). Any one of those three conditions is
enough reason to stop.

The final line does one more cleanup pass: `.filter()` drops any execution
missing a real start or end time (shouldn't normally happen, but it's cheap
insurance against a malformed record crashing something downstream),
`.sort()` puts everything in chronological order, and `.slice(0, limit)`
trims the final list back down to exactly the number requested — since the
last page fetched might have pushed us slightly over.

## Fetching executions across an entire project, concurrently

The "all pipelines in this project" feature (built starting in
[Chapter 20](20-building-the-filter-bar.md)) needs executions from *every*
pipeline in a project, not just one. The naive approach — call
`listExecutions` once per pipeline, one after another — would be needlessly
slow if a project has, say, thirty pipelines: each `await` would wait for
the previous one to fully finish before starting the next.

Instead, we run a bounded number of "workers" concurrently, each pulling
the next unclaimed pipeline off a shared list until none remain:

```ts
export type ProjectExecutions = {
  pipelineIdentifier: string;
  pipelineName: string;
  executions: ExecutionRecord[];
};

export async function listExecutionsForProject(
  org: string,
  project: string,
  pipelines: HarnessPipeline[],
  limitPerPipeline = 100,
): Promise<ProjectExecutions[]> {
  const CONCURRENCY = 6;
  const results: ProjectExecutions[] = new Array(pipelines.length);
  let index = 0;

  async function worker() {
    while (index < pipelines.length) {
      const current = index++;
      const p = pipelines[current];
      const executions = await listExecutions(org, project, p.identifier, limitPerPipeline, p.name);
      results[current] = { pipelineIdentifier: p.identifier, pipelineName: p.name, executions };
    }
  }

  await Promise.all(
    Array.from({ length: Math.min(CONCURRENCY, pipelines.length) }, worker),
  );

  return results;
}
```

This pattern is worth understanding slowly, since it reappears in
[Chapter 19](19-remaining-api-routes.md) for a different purpose:

- `let index = 0` is a shared counter, visible to every worker.
- Each `worker()` is a small async function that loops: grab the *current*
  value of `index`, then immediately increment it (`index++` — this
  happens synchronously, before any `await`, so two workers can never grab
  the same index), then `await` fetching that one pipeline's executions,
  store the result at that same position in the pre-sized `results` array,
  and loop back to check if there's more work.
- `Array.from({ length: Math.min(CONCURRENCY, pipelines.length) }, worker)`
  creates several independent workers — at most `CONCURRENCY` (6) of them,
  or fewer if there aren't even 6 pipelines to process — each one calling
  `worker()`, which returns a Promise (from
  [Chapter 4](04-javascript-crash-course.md)'s async/await coverage).
- `Promise.all(...)` waits for every one of those worker Promises to
  finish before this function returns.

The effect: at most 6 pipelines are ever being fetched from Harness at the
same time, but as soon as any one worker finishes its current pipeline, it
immediately grabs the next unclaimed one — so all 6 "lanes" stay busy until
the whole list is done, rather than sitting idle. This caps how many
simultaneous requests hit Harness's API (being a considerate API consumer,
and avoiding rate-limit errors) while still being dramatically faster than
doing pipelines strictly one at a time.

## Reading a child pipeline's outcome

Finally, the most detailed function in this file — fetching one child
pipeline's execution graph and pulling out the specific signals the
Option 1 investigation from [Chapter 12](12-the-option1-optimization-story.md)
needs. This one is long, so we'll build it in pieces.

First, two small helper functions for finding a specific node inside an
execution's graph by its "fully qualified name" (`baseFqn` — a
dot-separated path identifying exactly where in the pipeline's structure
this node sits, e.g. `pipeline.stages.Deploy.steps.Render_Manifest`):

```ts
function findNodeBySuffix(nodes: GraphNode[], suffix: string): GraphNode | undefined {
  return nodes.find((n) => n.baseFqn?.endsWith(suffix));
}

function findStageNode(nodes: GraphNode[], stageFqn: string): GraphNode | undefined {
  return nodes.find((n) => n.baseFqn === stageFqn);
}
```

Why two different matching strategies? `findNodeBySuffix` matches by
*ending* (`.endsWith(...)`) — useful for a step nested arbitrarily deep
inside a stage, where we only know its final segment
(`.steps.Render_Manifest`) and not everything before it. `findStageNode`
matches *exactly* — used for the two approval-gate stages
(`Approve_Deploy`, `Validate_Clusters`), whose own sub-steps have longer,
different-looking `baseFqn`s; an exact match on the stage's own known fqn
is unambiguous, where a suffix match risks accidentally grabbing the wrong
nested node if Harness's structure ever nests things differently. Choosing
the more precise tool where precision is available (and the more flexible
one where it's genuinely needed) is a small but real design decision.

Next, two more small helpers, converting a raw node into pieces of our
`OptimizationOutcome` type:

```ts
function toStepStatus(status: string | undefined): OptimizationStepStatus {
  if (status === "Skipped" || status === "Success" || status === "Failed") return status;
  return "Unknown";
}

function stepDurationMs(node: GraphNode | undefined): number | null {
  if (!node || !node.startTs || !node.endTs) return null;
  return node.endTs - node.startTs;
}

function stepWindow(node: GraphNode | undefined): { start: number | null; end: number | null } {
  if (!node || !node.startTs || !node.endTs) return { start: null, end: null };
  return { start: node.startTs, end: node.endTs };
}
```

`toStepStatus` narrows Harness's raw, unpredictable status string down to
our known `OptimizationStepStatus` union type from
[Chapter 16](16-defining-types.md) — anything unrecognized becomes
`"Unknown"` rather than crashing or letting an unexpected value leak into
the rest of the app. `stepDurationMs` and `stepWindow` both return `null`
(consistently, together — never one present without the other) whenever a
node wasn't found or is missing timing data, which is exactly the honest
"we don't know" modeling from Chapter 16 in action.

Now, the main function itself:

```ts
export async function getChildStageOutcome(
  stageName: string,
  parentExecutionId: string,
  childOrg: string,
  childProject: string,
  childPlanExecutionId: string,
): Promise<OptimizationOutcome | null> {
  if (useMockData()) return mockChildStageOutcome(stageName, parentExecutionId, childPlanExecutionId);

  const { accountId } = getConfig();
  const json = await harnessFetch(
    `/pipeline/api/pipelines/execution/v2/${childPlanExecutionId}?accountIdentifier=${accountId}&orgIdentifier=${childOrg}&projectIdentifier=${childProject}&renderFullBottomGraph=true`,
  );

  const nodeMap = json.data?.executionGraph?.nodeMap ?? {};
  const nodes: GraphNode[] = Object.values(nodeMap);

  const renderManifestNode = findNodeBySuffix(nodes, ".steps.Render_Manifest");
  const checkArgoNode = findNodeBySuffix(nodes, ".steps.Check_ArgoCD_Status");
  const deployAppNode = findNodeBySuffix(nodes, ".steps.Deploy_Application");
  const approveDeployNode = findStageNode(nodes, "pipeline.stages.Approve_Deploy");
  const validateClustersNode = findStageNode(nodes, "pipeline.stages.Validate_Clusters");

  if (!renderManifestNode && !checkArgoNode && !deployAppNode) return null;

  const manifestChangedRaw = renderManifestNode?.outcomes?.output?.outputVariables?.manifestChanged;
  const manifestChanged = manifestChangedRaw === undefined ? null : manifestChangedRaw === "true";

  const deployApplicationStatus = toStepStatus(deployAppNode?.status);
  const approveDeployWindow = stepWindow(approveDeployNode);
  const validateClustersWindow = stepWindow(validateClustersNode);

  return {
    stageName,
    parentExecutionId,
    childPlanExecutionId,
    manifestChanged,
    checkArgoStatus: toStepStatus(checkArgoNode?.status),
    checkArgoDurationMs: stepDurationMs(checkArgoNode),
    deployApplicationStatus,
    deployApplicationDurationMs: stepDurationMs(deployAppNode),
    optimizationFired: deployApplicationStatus === "Skipped",
    approveDeployStatus: toStepStatus(approveDeployNode?.status),
    approveDeployDurationMs: stepDurationMs(approveDeployNode),
    approveDeployStartTs: approveDeployWindow.start,
    approveDeployEndTs: approveDeployWindow.end,
    validateClustersStatus: toStepStatus(validateClustersNode?.status),
    validateClustersDurationMs: stepDurationMs(validateClustersNode),
    validateClustersStartTs: validateClustersWindow.start,
    validateClustersEndTs: validateClustersWindow.end,
  };
}
```

A few final things worth calling out:

- **The `?renderFullBottomGraph=true` query parameter** tells Harness to
  include the full, deeply nested step-by-step graph in its response — by
  default, an execution's API response might only summarize its top-level
  stages, not every nested step. We need the detail, so we ask for it
  explicitly.
- **The early `if (!renderManifestNode && !checkArgoNode && !deployAppNode) return null;`**
  check means: if none of the three core signal steps were found at all,
  this child pipeline execution simply doesn't use the Option 1 pattern —
  return `null` rather than a mostly-empty, misleading object. Whatever
  calls this function needs to handle that `null` case explicitly (you'll
  see this in [Chapter 19](19-remaining-api-routes.md)).
- **`manifestChangedRaw === undefined ? null : manifestChangedRaw === "true"`**
  handles a subtlety: the raw value comes back as the *string* `"true"` or
  `"false"` (a quirk of how Harness represents output variables), not an
  actual boolean. We convert it to a real `boolean`, but only if the field
  was present at all — if it's `undefined` (the step never ran, say), we
  correctly propagate that as `null` rather than guessing `false`.
- **`optimizationFired: deployApplicationStatus === "Skipped"`** is the
  actual measurement Chapter 12 promised: whether the optimization "fired"
  for this specific execution is defined as, and only as, whether
  `Deploy_Application`'s status came back `"Skipped"` — a single, precise,
  unambiguous condition, derived from data we've already carefully
  extracted above.

## Checkpoint

- [ ] `lib/harness.ts` exists, importing types from `lib/types.ts`, and
      exports `listOrgs`, `listProjects`, `isPpv3Pipeline`, `listPipelines`,
      `listExecutions`, `listExecutionsForProject`, and
      `getChildStageOutcome`.
- [ ] `npx tsc --noEmit` passes with no errors.
- [ ] You can explain, in your own words, why `listExecutionsForProject`
      uses a worker-pool pattern instead of either (a) fetching every
      pipeline one at a time, or (b) starting every fetch all at once with
      no limit.

**This generalizes to:** whatever external API your next dashboard talks
to — GitHub, a weather service, your company's internal API — build one
file exactly like this: a shared request helper handling auth and error
formatting once, a small `useMockData()`-style escape hatch if you want
offline development, and one clearly-named async function per operation,
each one translating that API's raw, possibly awkward response shape into
your own clean types from day one. Nothing else in your app should ever
need to know that API's URL structure or response format.

Next: [Chapter 18 — Your First API Route](18-first-api-route.md)
