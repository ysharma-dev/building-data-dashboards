---
layout: chapter
title: "Chapter 29 — Building Your Own Dashboard"
nav_order: 30
permalink: /29-building-your-own-dashboard/
---

# Chapter 29 — Building Your Own Dashboard

You've built a complete, real dashboard app — one that talks to a live
external API, models its data with types, computes honest metrics,
compares cohorts, runs a detailed investigation into one specific
hypothesis, charts trends, exports a shareable PDF, and is deployed at a
real URL. If you've made it here by actually typing the code rather than
skimming, you now know something more valuable than "how Harness Deploy
Insights works": you know the general shape every dashboard is built from,
because you've built one, piece by piece, and named the general skill
behind each piece as you went.

This chapter is where that transfers. First, a reference card mapping every
piece of what you built to the chapter that taught it. Then, three fully
worked-through alternative dashboard ideas — not vague suggestions, but
concrete enough to actually start building from tomorrow.

## The pattern reference card

Recall the "anatomy of a dashboard app" table from
[Chapter 0](00-introduction.md). Here it is again, now with every piece
tied to exactly where you built it — bookmark this page; you'll want to
come back to it the moment you start your next project.

| # | Piece | What it does | Where you built it |
|---|-------|--------------|---------------------|
| 1 | **Data source** | Where the raw information lives | [Ch. 10](10-harness-and-cd-pipelines.md) — how to read an unfamiliar API's data model |
| 2 | **Typed model** | Describes the shape of that data | [Ch. 16](16-defining-types.md) — `lib/types.ts` |
| 3 | **Fetch layer** | Knows how to ask the data source for data | [Ch. 17](17-talking-to-the-harness-api.md) — `lib/harness.ts` |
| 4 | **API routes** | Your own backend, between your UI and the data source | [Ch. 18](18-first-api-route.md), [Ch. 19](19-remaining-api-routes.md) — `app/api/**/route.ts` |
| 5 | **Filter UI** | Lets someone narrow what they're looking at | [Ch. 20](20-building-the-filter-bar.md) — the cascading filter bar |
| 6 | **Metrics module** | Turns raw records into meaningful numbers | [Ch. 11](11-metrics-explained.md) (the concepts) + [Ch. 22](22-computing-metrics.md) (the code) — `lib/dora.ts` |
| 7 | **Comparison view** | Puts two subsets side by side | [Ch. 23](23-ppv3-comparison.md) — the PPv3 comparison |
| 8 | **Deep-dive view** | A focused investigation of one specific question | [Ch. 12](12-the-option1-optimization-story.md) (the concepts) + [Ch. 24](24-the-option1-deep-dive.md) (the code) — the Option 1 analysis |
| 9 | **Charts** | Visual trends and comparisons | [Ch. 25](25-charts-with-recharts.md) — scatter, bar, and line charts |
| 10 | **Export** | Turning the view into something shareable | [Ch. 26](26-exporting-a-pdf-report.md) — the PDF export button |
| 11 | **Deployment** | Making the app available to other people | [Ch. 28](28-deploying-your-app.md) — Vercel |

Supporting all of this: the environment/secrets discipline from
[Chapter 15](15-environment-variables-and-secrets.md), the three-state
loading/error/data pattern from [Chapter 21](21-fetching-and-displaying-executions.md),
and the four recurring bug classes from
[Chapter 27](27-polish-accessibility-and-bugfixes.md) — none of those are
one specific "piece" of the anatomy table, but every piece above leans on
all three.

## How to use this card on a new project

When you sit down to build a dashboard on a completely different subject,
work through this table top to bottom, deciding for each row: **does this
piece transfer unchanged, transfer with small edits, or need to be
rebuilt from scratch for this subject?**

As a rule of thumb, gathered from the worked examples below:

- **Almost always rebuilt from scratch:** Pieces #1-3 (data source, typed
  model, fetch layer) and #6 (metrics module) — these are inherently
  specific to whatever you're measuring. A weather API's data model has
  nothing in common with Harness's, and "average temperature deviation"
  is a different calculation from "change failure rate." But the *process*
  you use to design each of these — read the API's docs and identify its
  entity hierarchy ([Chapter 10](10-harness-and-cd-pipelines.md)), model
  those entities as types before writing any code
  ([Chapter 16](16-defining-types.md)), isolate all API-specific knowledge
  in one fetch-layer file ([Chapter 17](17-talking-to-the-harness-api.md)),
  and design each metric as a rate, a ratio, or an honestly-labeled proxy
  ([Chapter 11](11-metrics-explained.md)) — carries over completely
  unchanged.
- **Usually reusable with light editing:** Pieces #4 (API routes — the
  same thin wrapper shape, [Chapter 18](18-first-api-route.md)), #5
  (filter UI — the same cascading pattern, just against a different
  hierarchy, [Chapter 20](20-building-the-filter-bar.md)), #9 (charts —
  the same handful of Recharts building blocks,
  [Chapter 25](25-charts-with-recharts.md)), and #10 (export — the same
  html2canvas-pro/jsPDF pipeline, [Chapter 26](26-exporting-a-pdf-report.md)).
- **Almost entirely reusable, verbatim:** the shadcn/ui component library
  and Tailwind design tokens from [Chapter 14](14-configuring-tailwind-and-shadcn.md),
  the Next.js project scaffolding from [Chapter 13](13-project-setup.md),
  and the deployment process from [Chapter 28](28-deploying-your-app.md).
  None of that has anything to do with deployments specifically — it's
  just "how to set up and ship a modern web app."

Now let's actually apply this to three different subjects — in real
detail, following that exact reasoning row by row.

## Project idea 1: a personal GitHub activity dashboard

**The question it answers:** "How active have I actually been on my own
open-source repositories, and is my contribution pattern healthy or
bursty?"

**Data source (Piece #1):** the [GitHub REST API](https://docs.github.com/en/rest),
specifically the `/user/repos` endpoint (list your repositories), the
`/repos/{owner}/{repo}/commits` endpoint (list commits on a repository, each
with an author and a timestamp), and `/repos/{owner}/{repo}/issues` and
`/repos/{owner}/{repo}/pulls` (issues and pull requests, each with a
`state` — `open` or `closed` — and timestamps for creation and closing).
Authentication works nearly identically to Harness: a personal access
token, sent as a header (`Authorization: Bearer <token>`), read from
`.env.local` exactly per [Chapter 15](15-environment-variables-and-secrets.md).

**Typed model (Piece #2):** mirroring [Chapter 16](16-defining-types.md)'s
approach — start from the entity hierarchy (a GitHub Organization or User
→ Repository → Commit / Issue / Pull Request), and design types like:

```ts
export type GitHubRepo = {
  id: string;
  name: string;
  ownerLogin: string;
};

export type CommitRecord = {
  repoName: string;
  sha: string;
  authorLogin: string | null;
  committedAt: number;
};

export type IssueRecord = {
  repoName: string;
  number: number;
  state: "open" | "closed";
  createdAt: number;
  closedAt: number | null;
  isPullRequest: boolean;
};
```

Notice `authorLogin: string | null` — GitHub's API can return a commit
with no linked GitHub account for its author (a commit made with an email
address that isn't associated with any account), exactly the same "model
a genuinely-missing value honestly with `| null`" lesson from
`OptimizationOutcome` in Chapter 16.

**Fetch layer (Piece #3):** a `lib/github.ts`, structured exactly like
`lib/harness.ts` — one shared `githubFetch()` helper handling the auth
header and error formatting (mirroring `harnessFetch` from
[Chapter 17](17-talking-to-the-harness-api.md)), and one exported async
function per operation: `listRepos()`, `listCommits(repo)`,
`listIssues(repo)`.

**API routes (Piece #4):** `app/api/repos/route.ts`,
`app/api/commits/route.ts`, `app/api/issues/route.ts` — the same thin
`GET` wrapper shape from [Chapter 18](18-first-api-route.md), since a
GitHub personal access token is exactly the kind of secret that must never
reach browser code.

**Metrics module (Piece #6), with real formulas:**
- **Commits per week** (a RATE, same shape as Deployment Frequency):
  count of commits ÷ span in weeks between your oldest and newest fetched
  commit.
- **Issue close rate** (a RATIO, same shape as Change Failure Rate): count
  of issues with `state === "closed"` ÷ total issue count.
- **Average time-to-close** (a genuine, *non-proxy* metric here, unlike
  Harness's Lead Time — GitHub's own timestamps directly give you
  `createdAt` and `closedAt` on the same record, no cross-system proxy
  needed): average of `closedAt - createdAt` across closed issues.
- **Longest streak without a commit** — a new kind of metric this project
  didn't need: sort commits by date, and find the largest gap between two
  consecutive commit dates. A nice example of a metric that isn't a rate,
  a ratio, or a proxy — sometimes the right metric is neither, and that's
  fine; the three shapes from [Chapter 11](11-metrics-explained.md) are
  the *common* cases, not the *only* ones.

**Filter hierarchy (Piece #5):** Repository → (optionally) Author, since
one repository can have contributions from several people — a shallower,
two-level cascade compared to this book's three-level Org → Project →
Pipeline, built the exact same way as [Chapter 20](20-building-the-filter-bar.md).

**Deep-dive idea (Piece #8):** "Did switching to a new branching strategy
on <date> actually reduce how long pull requests stay open?" — the exact
same shape as the Option 1 investigation: a rollout cutoff date, a
before/after comparison of average PR-open-duration, and a confound to
watch for (a pull request that sat open for months because of an
unrelated, ongoing design discussion — not because your new process is
slow — is exactly the kind of "human wait time" confound that
[Chapter 12](12-the-option1-optimization-story.md)'s Approve Deploy gate
taught you to isolate rather than silently average in).

## Project idea 2: a personal weather/climate trends dashboard

**The question it answers:** "How has the weather in my city actually
changed over the past several years, compared to what I remember?"

**Data source (Piece #1):** a free weather API such as
[Open-Meteo](https://open-meteo.com/) (no API key required at all,
actually simplifying [Chapter 15](15-environment-variables-and-secrets.md)'s
secrets-handling — a good first project if you want to skip straight past
the API-key setup) or a similar historical-weather API, returning daily
records with a date, a high/low temperature, and precipitation.

**Typed model (Piece #2):**

```ts
export type DailyWeatherRecord = {
  date: string;
  highTempC: number;
  lowTempC: number;
  precipitationMm: number;
};
```

Notably simpler than `ExecutionRecord` — there's no equivalent of nested
`stages` or `childStageLinks` here, since a daily weather reading doesn't
have an internal step-by-step structure the way a pipeline execution does.
Not every dashboard's core entity needs to be as structurally rich as this
book's `ExecutionRecord` — model exactly the fields your subject actually
has, no more.

**Metrics module (Piece #6), with real formulas:**
- **Average high temperature per month** — a rate-shaped calculation
  (grouped by month instead of divided by a time span, but the same
  "bucket by time period, then average within each bucket" idea as the
  weekly-binning logic in [Chapter 25](25-charts-with-recharts.md)'s line
  chart).
- **Days above/below a threshold** (a RATIO) — count of days where
  `highTempC` exceeds, say, 35°C ÷ total days in the period.
- **Year-over-year deviation** (a genuinely new shape): for each
  day-of-year, compare this year's high temperature against the average
  of the same day-of-year across all previous years in your dataset — a
  "compare against a historical baseline" metric, a shape this book's DORA
  metrics never needed, since Harness Deploy Insights never compares
  "this year" against "past years" for the same calendar day.

**Filter hierarchy (Piece #5):** City → Year — a good example of a filter
hierarchy that isn't "broader group containing narrower group" in quite
the same organizational sense as Org → Project, but works via the exact
same cascading mechanism from [Chapter 20](20-building-the-filter-bar.md):
picking a city determines which years of data are even available to
select next.

**Deep-dive idea (Piece #8):** "Has this specific month gotten
measurably hotter over the past decade, once you exclude anomalous single
-day heat spikes?" — cutoff-date-style filtering (comparing "the last 5
years" against "the 5 years before that," rather than a strict
before/after rollout split), and a genuine confound to isolate: a single
extreme heatwave day can skew a monthly average dramatically, the same way
a single slow, human-approval-gated deployment could skew an average
deploy duration in this book's app — the fix is the same idea taught in
[Chapter 12](12-the-option1-optimization-story.md) and
[Chapter 24](24-the-option1-deep-dive.md): decide explicitly whether
outliers get included, excluded, or shown separately, and say which,
rather than letting them silently distort an average without comment.

## Project idea 3: a personal finance / spending dashboard

**The question it answers:** "Where does my money actually go each month,
and is my spending trending up or down in categories I care about?"

**Data source (Piece #1):** a CSV export from your bank or budgeting app
— a genuinely different *kind* of data source from the first two ideas
(a file you upload, rather than a live API you call), which is a useful
variation to practice: instead of a `lib/*.ts` fetch layer calling
`fetch()` against a remote URL, you'd write a parsing function that reads
an uploaded CSV file's rows and turns each one into a typed record — the
exact same *idea* as Piece #3 (isolate all knowledge of "how do I get data
out of this source" in one place), applied to a file instead of a network
request.

**Typed model (Piece #2):**

```ts
export type Transaction = {
  id: string;
  date: number;
  merchant: string;
  category: string;
  amountCents: number;
};
```

`amountCents` (rather than a floating-point dollar amount) is a genuinely
useful, non-obvious detail worth calling out: storing money as whole
integer cents avoids floating-point rounding errors that can accumulate
when you add up many decimal dollar amounts — a real-world lesson in
choosing a data type that matches what you're actually going to do with
the value (sum many of them, precisely), the same spirit as
[Chapter 16](16-defining-types.md)'s "design the type to match how the
value will actually be used."

**Metrics module (Piece #6), with real formulas:**
- **Total spend per category per month** — a straightforward grouped sum
  (group transactions by category and month, using the exact
  get-with-fallback-mutate-put-back Map pattern from
  [Chapter 22](22-computing-metrics.md)'s `computeMttrProxy`, then sum
  `amountCents` within each group).
- **Percentage of income saved** (a RATIO) — (income transactions total −
  expense transactions total) ÷ income transactions total.
- **Biggest month-over-month category increase** — sort each category's
  month-over-month percentage change and surface the largest one, a
  "find the extreme, not the average" metric — another example, like the
  weather dashboard's longest-gap-without-a-commit, of a useful metric
  that isn't a rate, ratio, or proxy.

**Filter hierarchy (Piece #5):** Account → Category — again a two-level
cascade, built identically to [Chapter 20](20-building-the-filter-bar.md).

**Deep-dive idea (Piece #8):** "Did a specific budgeting change I made on
<date> actually reduce discretionary spending?" — a before/after
comparison exactly like Option 1's, with an important confound to isolate:
a one-time large purchase (a new laptop, a vacation) shouldn't be
attributed to your budgeting habit changing for better or worse — the
same "isolate a wait-time confound so it doesn't distort the deploy-
mechanism numbers" discipline from
[Chapter 12](12-the-option1-optimization-story.md), applied to "isolate a
one-time purchase so it doesn't distort your habitual-spending numbers,"
shown in its own clearly-labeled "large one-time purchases, excluded
above" panel rather than silently included or silently deleted.

## What to actually reuse, file for file

For any of the three projects above (or your own idea), here's what
genuinely carries over with little to no change, versus what you'll write
fresh:

**Copy over almost unchanged:**
- Every file under `components/ui/` (the shadcn primitives from
  [Chapter 14](14-configuring-tailwind-and-shadcn.md)) — a button is a
  button, regardless of subject.
- `lib/utils.ts` (the `cn()` helper) and `lib/interval-merge.ts` (if your
  deep-dive involves any overlapping time windows at all — genuinely
  domain-agnostic, general-purpose interval math).
- `components/info-tip.tsx` and `components/truncated-text.tsx` — both
  entirely generic UI helpers with no Harness-specific content at all.
- The overall structure of `app/layout.tsx`, `app/globals.css`'s token
  setup, and every config file from
  [Chapter 13](13-project-setup.md)/[14](14-configuring-tailwind-and-shadcn.md).
- The deployment process from [Chapter 28](28-deploying-your-app.md), start
  to finish.

**Rewrite the shape of, not the content:**
- `lib/types.ts`, your fetch-layer file, your API routes, your metrics
  module, your filter bar's specific field names, your chart's specific
  data — every one of these keeps the *pattern* this book taught, with
  entirely new field names, entity names, and formulas underneath.

**Genuinely new problems worth expecting:**
- Every project above surfaced at least one metric that isn't a clean
  rate/ratio/proxy (a longest-gap, a year-over-year deviation, a
  biggest-increase) — a reminder that
  [Chapter 11](11-metrics-explained.md)'s three shapes are a strong
  starting toolkit, not an exhaustive list. When your subject calls for
  something else, design it with the same care (define it precisely,
  double check it against a worked numeric example, watch for confounds)
  even if it doesn't fit one of the three named shapes.

## Closing thought

The specific numbers in Harness Deploy Insights — deployment frequency,
skip rates, approval wait times — will mean nothing to you a year from
now if deployments aren't part of your work. But "read the API's shape
before writing code," "model data with honest types, `null` included,"
"isolate every external API's quirks behind one small file," "label a
proxy honestly," "hunt for the confound before trusting a number," "reuse
one display component across every subset of data instead of duplicating
it," and "never let an empty result quietly look like a measured zero" —
those will keep being useful for as long as you build software that shows
people what's happening with their data. That's the actual thing this book
was teaching. Go build something with it.

Next: if you skipped the Harness account setup, see
[Appendix A](appendix-a-mock-data-fallback.md) for the mock-data path;
otherwise, [Appendix B](appendix-b-troubleshooting.md) and
[Appendix C](appendix-c-glossary.md) are there whenever you need them.
