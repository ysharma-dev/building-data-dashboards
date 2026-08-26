# Chapter 19 — The Remaining API Routes

**Skill:** recognizing when a new piece of code is "the same pattern again"
versus genuinely new — so you can move fast through repetition and slow
down exactly where it matters. Most of what follows in this chapter is
copy the shape, change the details; one part genuinely isn't, and knowing
which is which is itself a skill.

## The repeating shape

You just built `app/api/orgs/route.ts` in
[Chapter 18](18-first-api-route.md). Four more routes in this app follow
the *exact* same shape: read query parameters from the request, validate
that the required ones are present, call the matching function from
`lib/harness.ts`, and wrap the result (or an error) as JSON. Let's move
through them quickly, since the pattern itself is already familiar — and
call out only what's actually new in each one.

### app/api/projects/route.ts

```ts
import { NextResponse } from "next/server";
import { listProjects } from "@/lib/harness";

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const org = searchParams.get("org");

  if (!org) {
    return NextResponse.json({ error: "Missing org param" }, { status: 400 });
  }

  try {
    const projects = await listProjects(org);
    return NextResponse.json(projects);
  } catch (err) {
    return NextResponse.json(
      { error: err instanceof Error ? err.message : "Unknown error" },
      { status: 400 },
    );
  }
}
```

**What's new here:** unlike `GET /api/orgs`, this route needs an input —
*which* org's projects to list. That input arrives as a **query
parameter**: the part of a URL after a `?`, like
`/api/projects?org=my-org-id`. `new URL(request.url).searchParams` parses
those out into something you can `.get("org")` from, by name.

Notice the explicit `if (!org)` check *before* even trying to call
`listProjects`. This is a second, distinct kind of error from the
`try`/`catch` below it: this one is the caller's fault (they forgot to
include a required parameter), so it gets its own clear message and the
HTTP status `400` ("Bad Request" — meaning "the request itself was
malformed," as opposed to `500`, "something broke on our end while
processing an otherwise-valid request"). Distinguishing "you sent me
something invalid" from "I failed while handling something valid" is a
genuinely useful habit — it helps whoever's debugging a failed request
immediately know which side of the conversation to look at first.

### app/api/pipelines/route.ts

```ts
import { NextResponse } from "next/server";
import { listPipelines } from "@/lib/harness";

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const org = searchParams.get("org");
  const project = searchParams.get("project");

  if (!org || !project) {
    return NextResponse.json({ error: "Missing org or project param" }, { status: 400 });
  }

  try {
    const pipelines = await listPipelines(org, project);
    return NextResponse.json(pipelines);
  } catch (err) {
    return NextResponse.json(
      { error: err instanceof Error ? err.message : "Unknown error" },
      { status: 500 },
    );
  }
}
```

**What's new:** nothing conceptually — just one more query parameter
(`project`, alongside `org`), since listing pipelines requires knowing
which project you're inside. Same shape, one more required input.

### app/api/executions/route.ts

```ts
import { NextResponse } from "next/server";
import { listExecutions } from "@/lib/harness";

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const org = searchParams.get("org");
  const project = searchParams.get("project");
  const pipeline = searchParams.get("pipeline");
  const pipelineName = searchParams.get("pipelineName") ?? undefined;
  const limit = Number(searchParams.get("limit") ?? "100");

  if (!org || !project || !pipeline) {
    return NextResponse.json(
      { error: "Missing org, project, or pipeline param" },
      { status: 400 },
    );
  }

  try {
    const executions = await listExecutions(org, project, pipeline, limit, pipelineName);
    return NextResponse.json(executions);
  } catch (err) {
    return NextResponse.json(
      { error: err instanceof Error ? err.message : "Unknown error" },
      { status: 500 },
    );
  }
}
```

**What's new:** two optional parameters, handled slightly differently from
the required ones above. `pipelineName` uses `?? undefined` — if it's
missing from the URL, we pass `undefined` along, and recall from
[Chapter 17](17-talking-to-the-harness-api.md) that `listExecutions` already
has a default parameter (`pipelineName: string = pipeline`) that kicks in
when `undefined` is passed. `limit` similarly defaults to `"100"` as a
string before being converted to an actual number with `Number(...)` —
query parameters are *always* strings, even when they represent numbers, so
converting explicitly is necessary here in a way it wasn't for `org` or
`project`, which are used as strings anyway.

### app/api/executions/project/route.ts

```ts
import { NextResponse } from "next/server";
import { listExecutionsForProject, listPipelines } from "@/lib/harness";

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const org = searchParams.get("org");
  const project = searchParams.get("project");
  const limitPerPipeline = Number(searchParams.get("limitPerPipeline") ?? "50");

  if (!org || !project) {
    return NextResponse.json({ error: "Missing org or project param" }, { status: 400 });
  }

  try {
    const pipelines = await listPipelines(org, project);
    const results = await listExecutionsForProject(org, project, pipelines, limitPerPipeline);
    return NextResponse.json(results);
  } catch (err) {
    return NextResponse.json(
      { error: err instanceof Error ? err.message : "Unknown error" },
      { status: 500 },
    );
  }
}
```

Two things worth pointing out here:

- **The file lives at `app/api/executions/project/route.ts`** — a *nested*
  folder under `executions/`. Remember, folder structure directly becomes
  URL structure in the App Router, so this becomes `/api/executions/project`
  — a sibling endpoint to `/api/executions`, not a parameter of it. This is
  a clean way to represent "the same general concept (executions), but a
  meaningfully different query (all pipelines in a project, rather than one
  specific pipeline)" as two distinct URLs rather than cramming both
  behaviors into one endpoint with a confusing flag parameter.
- **This route calls two different `lib/harness.ts` functions in
  sequence** — first `listPipelines` (to discover every pipeline in the
  project), then feeds that result into `listExecutionsForProject` (the
  concurrent worker-pool function from
  [Chapter 17](17-talking-to-the-harness-api.md)). This is a small but
  genuine example of an API route doing more than a single 1-to-1 pass
  through to `lib/harness.ts` — it's composing two lower-level operations
  into one higher-level one, which is exactly the kind of thing an API
  route layer is *for*.

## The one route that's actually different: the batch endpoint

Now the part that genuinely isn't "the same pattern again." Every route so
far has been a `GET` — "give me data based on some parameters in the URL."
The Option 1 investigation from [Chapter 12](12-the-option1-optimization-story.md)
needs something different: given a *list* of child pipeline links (which
can be dozens or hundreds of them, gathered from many executions at once),
fetch the outcome for *every one of them*, capped at a reasonable
concurrency so we don't hammer Harness's API with hundreds of simultaneous
requests.

This needs a `POST` request instead of `GET`, because we're sending a
sizable list of structured data *to* the server, not just a few short
values that fit comfortably in a URL's query string.

```ts
import { NextResponse } from "next/server";
import { getChildStageOutcome } from "@/lib/harness";

type BatchItem = {
  stageName: string;
  parentExecutionId: string;
  childOrg: string;
  childProject: string;
  childPlanExecutionId: string;
};

const CONCURRENCY = 8;

async function mapWithConcurrency<T, R>(
  items: T[],
  limit: number,
  fn: (item: T) => Promise<R>,
): Promise<R[]> {
  const results: R[] = new Array(items.length);
  let index = 0;

  async function worker() {
    while (index < items.length) {
      const current = index++;
      results[current] = await fn(items[current]);
    }
  }

  await Promise.all(Array.from({ length: Math.min(limit, items.length) }, worker));
  return results;
}

export async function POST(request: Request) {
  let items: BatchItem[];
  try {
    items = await request.json();
  } catch {
    return NextResponse.json({ error: "Invalid JSON body" }, { status: 400 });
  }

  if (!Array.isArray(items)) {
    return NextResponse.json({ error: "Body must be an array" }, { status: 400 });
  }

  try {
    const outcomes = await mapWithConcurrency(items, CONCURRENCY, async (item) => {
      try {
        return await getChildStageOutcome(
          item.stageName,
          item.parentExecutionId,
          item.childOrg,
          item.childProject,
          item.childPlanExecutionId,
        );
      } catch {
        return null;
      }
    });

    return NextResponse.json(outcomes.filter(Boolean));
  } catch (err) {
    return NextResponse.json(
      { error: err instanceof Error ? err.message : "Unknown error" },
      { status: 500 },
    );
  }
}
```

Let's take this apart, since there's real, new, generalizable content here.

**`mapWithConcurrency` is a *generic* function** — notice the
`<T, R>` in its signature, a TypeScript feature briefly mentioned in
[Chapter 5](05-typescript-crash-course.md). It doesn't hardcode "a list of
`BatchItem`" or "a list of `OptimizationOutcome`" — it works with *any*
array of items (`T`) and *any* async function that turns one item into some
result (`R`), and returns an array of those results. This is exactly the
same worker-pool pattern as `listExecutionsForProject` from
[Chapter 17](17-talking-to-the-harness-api.md) — a shared `index` counter,
several `worker()` functions racing to claim the next unclaimed item,
`Promise.all` waiting for them all to finish — except written once,
generically, so it can be reused here for a completely different kind of
item. Recognizing "I've already solved this exact shape of problem
elsewhere" and extracting it into a reusable, generic helper — rather than
copy-pasting the loop again with different variable names — is a real skill
worth practicing, and this is a clean, small example of it.

**Reading the request body.** Unlike a `GET` route reading query
parameters from the URL, a `POST` route reads its input from the request's
*body* — the actual payload sent along with the request, which for a JSON
API is (unsurprisingly) JSON text. `await request.json()` parses that text
into a real JavaScript value. Wrapping this in its own `try`/`catch` (with
a distinct `400` "Invalid JSON body" error) handles the case where whatever
called this route sent malformed or non-JSON text — a different failure
mode from "the request was well-formed JSON, but shaped wrong," which the
next check (`!Array.isArray(items)`) catches separately.

**Per-item error isolation.** Notice the *inner* `try`/`catch`, wrapping
just the single call to `getChildStageOutcome` inside the concurrency
worker, returning `null` on failure rather than letting one item's error
propagate up and fail the *entire* batch. This is a deliberate and
important design decision: if one specific child pipeline execution has
some malformed or unreachable data, that shouldn't take down the analysis
for the other ninety-nine items in the batch. `outcomes.filter(Boolean)` at
the end then drops every `null` (both from a `getChildStageOutcome` call
that itself intentionally returns `null`, per
[Chapter 17](17-talking-to-the-harness-api.md), and from ones that errored
here) — the caller on the other end simply receives however many outcomes
were successfully retrieved, silently skipping the rest, rather than an
all-or-nothing failure.

This "isolate failures per-item in a batch, rather than letting one bad
item sink the whole operation" pattern is broadly useful anywhere you're
processing a list of independent things where partial success is more
valuable than all-or-nothing failure.

## Testing the batch route with curl

Since this one takes a `POST` request with a JSON body, testing it with
curl looks slightly different from Chapter 18's simple `GET`:

```bash
curl -X POST http://localhost:3000/api/optimization/batch \
  -H "Content-Type: application/json" \
  -d '[{"stageName":"Deploy","parentExecutionId":"abc","childOrg":"default","childProject":"demo","childPlanExecutionId":"xyz"}]'
```

`-X POST` sets the HTTP method, `-H` sets a header (telling the server
the body is JSON), and `-d` provides the request body itself. With mock
data enabled, you should get back a JSON array with one outcome object in
it — confirming the whole POST → parse-body → process → respond path works,
still with zero UI involved.

## Checkpoint

- [ ] All five remaining routes exist: `app/api/projects/route.ts`,
      `app/api/pipelines/route.ts`, `app/api/executions/route.ts`,
      `app/api/executions/project/route.ts`, and
      `app/api/optimization/batch/route.ts`.
- [ ] Each `GET` route correctly returns a `400` error when a required
      query parameter is missing (test by leaving one out of the URL).
- [ ] The `curl -X POST` command above returns a JSON array, not an error.
- [ ] You can explain, in your own words, why one bad item in a batch
      request shouldn't fail the entire batch.

**This generalizes to:** most of any API layer you'll ever build is
"the same pattern again" — read input, validate it, call your data layer,
wrap the result — and recognizing that lets you move through it quickly and
consistently. But watch for the moments that genuinely aren't the pattern:
whenever you're processing *several* independent things in one request
(a batch import, a bulk update, a multi-file upload), isolate failures
per-item rather than letting one bad item fail everything, and reach for a
small, reusable, generic concurrency helper rather than writing a bespoke
loop every time you need to do several things "at once, but not too many at
once."

Next: [Chapter 20 — Building the Filter Bar](20-building-the-filter-bar.md)
