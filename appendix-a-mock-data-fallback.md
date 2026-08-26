# Appendix A — Running Without a Harness Account (Mock Data)

This appendix builds out the full mock-data path introduced in
[Chapter 15](15-environment-variables-and-secrets.md) and referenced
throughout [Chapter 17](17-talking-to-the-harness-api.md): a way to run
the entire app, with realistic-looking data, without ever creating a
Harness account or holding a real API key. If you've been following the
book with `USE_MOCK_DATA=true` in your `.env.local`, this appendix is
where that data actually comes from.

## Where this code lives

Create `lib/mock-data.ts`. This file's only job is to produce fake, but
internally consistent, data matching every type from
[Chapter 16](16-defining-types.md) — every function in `lib/harness.ts`
that has a mock branch calls into this file.

## Mock orgs, projects, and pipelines

Start simple — a small, fixed, hardcoded hierarchy, small enough to
browse comfortably but large enough to exercise every dropdown in the
filter bar from [Chapter 20](20-building-the-filter-bar.md):

```ts
import type { HarnessOrg, HarnessPipeline, HarnessProject } from "./types";

const MOCK_ORGS: HarnessOrg[] = [
  { identifier: "default", name: "Default Org" },
  { identifier: "platform", name: "Platform Team" },
];

const MOCK_PROJECTS: Record<string, HarnessProject[]> = {
  default: [
    { identifier: "demo_project", name: "Demo Project", orgIdentifier: "default" },
  ],
  platform: [
    { identifier: "infra", name: "Infrastructure", orgIdentifier: "platform" },
  ],
};

const MOCK_PIPELINES: Record<string, HarnessPipeline[]> = {
  demo_project: [
    { identifier: "checkout_service_ppv3", name: "Checkout Service (ppv3)" },
    { identifier: "billing_service", name: "Billing Service" },
  ],
  infra: [
    { identifier: "cluster_upgrade", name: "Cluster Upgrade" },
  ],
};

export function mockOrgs(): HarnessOrg[] {
  return MOCK_ORGS;
}

export function mockProjects(org: string): HarnessProject[] {
  return MOCK_PROJECTS[org] ?? [];
}

export function mockPipelines(org: string, project: string): HarnessPipeline[] {
  return MOCK_PIPELINES[project] ?? [];
}
```

Notice `checkout_service_ppv3` deliberately contains "ppv3" in its
identifier — this is what makes `isPpv3Pipeline` (from
[Chapter 17](17-talking-to-the-harness-api.md)) correctly classify it,
letting you exercise the PPv3 comparison view from
[Chapter 23](23-ppv3-comparison.md) against mock data too. A `Record<string, T[]>`
(a plain object used as a lookup table, keyed by string) is a simple,
readable way to hardcode "this parent has these children" relationships
without a database.

## Mock executions, generated rather than hardcoded

A hardcoded list of 3-4 fake executions wouldn't be enough to see anything
interesting in the DORA metrics or charts — you need enough spread over
enough time to make a trend visible. Rather than hand-writing dozens of
fake executions, generate them programmatically, with just enough
randomness to look realistic, but deterministic enough to be usable for
following along with the book (the same seed of "reasonable-looking data"
every time you restart your dev server):

```ts
import type { ChildStageLink, ExecutionRecord, StageDuration } from "./types";

const STATUSES: { status: string; weight: number }[] = [
  { status: "Success", weight: 8 },
  { status: "Failed", weight: 1 },
  { status: "Aborted", weight: 1 },
];

function pickWeightedStatus(rng: () => number): string {
  const total = STATUSES.reduce((sum, s) => sum + s.weight, 0);
  let roll = rng() * total;
  for (const s of STATUSES) {
    if (roll < s.weight) return s.status;
    roll -= s.weight;
  }
  return "Success";
}

// A small, seeded pseudo-random number generator — NOT for anything
// security-sensitive, only so mock data looks the same across restarts
// instead of changing every time you reload the page.
function seededRandom(seed: number): () => number {
  let state = seed;
  return () => {
    state = (state * 1103515245 + 12345) % 2147483648;
    return state / 2147483648;
  };
}

export function mockExecutions(
  pipeline: string,
  pipelineName: string,
  limit: number,
): ExecutionRecord[] {
  const rng = seededRandom(pipeline.length * 7919);
  const now = 1_735_000_000_000; // a fixed reference "now" for reproducibility
  const count = Math.min(limit, 60);
  const executions: ExecutionRecord[] = [];

  for (let i = 0; i < count; i++) {
    const daysAgo = count - i;
    const startTs = now - daysAgo * 24 * 60 * 60 * 1000 + Math.floor(rng() * 3_600_000);
    const baseDurationMs = 8 * 60 * 1000 + Math.floor(rng() * 6 * 60 * 1000);
    const status = pickWeightedStatus(rng);
    const durationMs = status === "Aborted" ? baseDurationMs + Math.floor(rng() * 20 * 60 * 1000) : baseDurationMs;
    const endTs = startTs + durationMs;

    const stages: StageDuration[] = [
      { name: "Build", durationMs: Math.floor(durationMs * 0.3) },
      { name: "Deploy", durationMs: Math.floor(durationMs * 0.7) },
    ];

    const childStageLinks: ChildStageLink[] = pipeline.includes("ppv3")
      ? [
          {
            stageName: "Deploy",
            childPlanExecutionId: `${pipeline}-child-${i}`,
            childOrgId: "default",
            childProjectId: "demo_project",
          },
        ]
      : [];

    executions.push({
      id: `${pipeline}-exec-${i}`,
      status,
      startTs,
      endTs,
      durationMs,
      stages,
      childStageLinks,
      pipelineIdentifier: pipeline,
      pipelineName,
    });
  }

  return executions;
}
```

A few things worth understanding about the choices here:

- **`seededRandom` exists because `Math.random()` is different every
  time.** For following along with a book, it's much more useful for your
  mock data to look *the same* every time you restart your dev server —
  the DORA metrics you compute today should still make sense tomorrow
  when you revisit the app, rather than reshuffling every reload. A seeded
  generator produces the same sequence of "random-looking" numbers every
  time, given the same starting seed.
- **`pickWeightedStatus` favors `"Success"`** (a weight of 8 out of a
  total of 10) over `"Failed"`/`"Aborted"` (1 each) — matching a
  realistic Change Failure Rate (recall
  [Chapter 11](11-metrics-explained.md)): most deploys should succeed, or
  the mock data would misleadingly suggest an unhealthy pipeline by
  default.
- **Only pipelines whose identifier contains `"ppv3"` get any
  `childStageLinks`** — mirroring the real app's actual behavior, where
  only pipelines built around the parent-calls-child pattern from
  [Chapter 12](12-the-option1-optimization-story.md) have any child links
  at all. This lets you see the Option 1 deep dive correctly show its
  "no child-pipeline calls found" empty state for `billing_service`, and
  real analysis for `checkout_service_ppv3` — exactly the same behavior
  you'd see against real data.

## Mock child-stage outcomes

Finally, mock data for the Option 1 investigation itself — recall
`getChildStageOutcome` from [Chapter 17](17-talking-to-the-harness-api.md):

```ts
import type { OptimizationOutcome } from "./types";

export function mockChildStageOutcome(
  stageName: string,
  parentExecutionId: string,
  childPlanExecutionId: string,
): OptimizationOutcome | null {
  const rng = seededRandom(childPlanExecutionId.length * 104729);
  const manifestChanged = rng() < 0.35; // ~35% of deploys actually change the manifest
  const deployApplicationStatus = manifestChanged ? "Success" : "Skipped";

  return {
    stageName,
    parentExecutionId,
    childPlanExecutionId,
    manifestChanged,
    checkArgoStatus: "Success",
    checkArgoDurationMs: 15_000 + Math.floor(rng() * 10_000),
    deployApplicationStatus,
    deployApplicationDurationMs: manifestChanged ? 90_000 + Math.floor(rng() * 60_000) : null,
    optimizationFired: !manifestChanged,
    approveDeployStatus: rng() < 0.5 ? "Success" : "Skipped",
    approveDeployDurationMs: rng() < 0.5 ? 300_000 + Math.floor(rng() * 900_000) : null,
    approveDeployStartTs: null,
    approveDeployEndTs: null,
    validateClustersStatus: "Skipped",
    validateClustersDurationMs: null,
    validateClustersStartTs: null,
    validateClustersEndTs: null,
  };
}
```

Notice `manifestChanged` is deliberately set to be `false` (meaning the
optimization *fires*, and the deploy step is *skipped*) roughly 65% of the
time — a reasonably high, "the optimization usually helps" skip rate, so
that clicking through to the Option 1 deep dive against mock data actually
shows a healthy, non-zero skip rate rather than a discouraging "it never
fires" result. `deployApplicationDurationMs` is explicitly `null` when the
step was skipped — the same "a skipped step genuinely has no real duration
to report" honesty from [Chapter 16](16-defining-types.md), preserved even
in fake data.

## Wiring it into `lib/harness.ts`

With `lib/mock-data.ts` written, go back to the `useMockData()` checks
from [Chapter 17](17-talking-to-the-harness-api.md) — each one should now
have a real function to call:

```ts
import { mockOrgs, mockProjects, mockPipelines, mockExecutions, mockChildStageOutcome } from "./mock-data";

export async function listOrgs(): Promise<HarnessOrg[]> {
  if (useMockData()) return mockOrgs();
  // ...real Harness call
}
```

## Checkpoint

- [ ] `lib/mock-data.ts` exists and exports `mockOrgs`, `mockProjects`,
      `mockPipelines`, `mockExecutions`, and `mockChildStageOutcome`.
- [ ] With `USE_MOCK_DATA=true` in `.env.local`, the app's filter bar
      populates with the mock orgs/projects/pipelines above, with no real
      network calls to Harness.
- [ ] Selecting `checkout_service_ppv3` and expanding the Option 1 deep
      dive shows a real skip rate, not an empty state.
- [ ] Reloading the page shows the *same* mock data as before (confirming
      `seededRandom` is working as a deterministic generator, not
      `Math.random()`).

This mock-data pattern is worth keeping as a habit in every future
project, not just as a workaround for this book: any time your app depends
on a paid or account-gated external API, a small `USE_MOCK_DATA`-style
escape hatch — with just enough realistic, seeded fake data to exercise
every feature — makes local development, demos, and even automated tests
dramatically easier, for you and for anyone else who ever needs to run
your project without live credentials in front of them.
