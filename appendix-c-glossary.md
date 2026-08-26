# Appendix C — Glossary

Every technical term used in this book, defined plainly, alphabetized. If
a term appears here that you don't remember encountering, that's fine —
use this as a lookup, not required reading.

**API (Application Programming Interface)** — a defined way for one piece
of software to ask another for data or to make it do something, without
needing to know how that other software works internally.

**API key** — a secret string that proves to a server "this request is
authorized, on behalf of this account." Must never be exposed in
browser-side code or committed to Git. See [Chapter 15](15-environment-variables-and-secrets.md).

**API route / Route Handler** — in Next.js, a file named `route.ts` under
`app/`, exporting functions like `GET` or `POST`, that becomes a backend
endpoint at that file's URL path. See [Chapter 7](07-what-is-nextjs.md),
[Chapter 18](18-first-api-route.md).

**App Router** — Next.js's current, folder-based routing system, using an
`app/` folder. Distinct from the older "Pages Router." See
[Chapter 7](07-what-is-nextjs.md).

**Async / await** — JavaScript syntax for writing code that waits for a
slow operation (like a network request) without freezing the whole
program while it waits. See [Chapter 4](04-javascript-crash-course.md).

**Base UI** — the headless (unstyled, behavior-only) component library
this project's shadcn/ui components are built on top of. See
[Chapter 9](09-what-is-shadcn-ui.md).

**Change Failure Rate** — one of the four DORA metrics: the percentage of
deployments that failed or were aborted. See [Chapter 11](11-metrics-explained.md).

**Client Component** — in Next.js, a component marked with `"use client"`
that runs in the browser and can use interactive features like `useState`.
Contrast with Server Component. See [Chapter 7](07-what-is-nextjs.md).

**Commit** — in Git, a saved snapshot of your project's files at a point
in time, with a message describing what changed. See
[Chapter 3](03-git-and-github-basics.md).

**Confound** — a factor mixed into a measurement that has nothing to do
with what you're actually trying to measure, and which can corrupt your
conclusion if not isolated or removed. See
[Chapter 12](12-the-option1-optimization-story.md).

**CORS (Cross-Origin Resource Sharing)** — a browser security mechanism
that restricts which websites can make requests to a given API on a
visitor's behalf. See [Chapter 18](18-first-api-route.md).

**Deep-dive view** — a dashboard feature offering a detailed, focused
investigation of one specific question, layered on top of basic filtering
and metrics. Piece #8 of the anatomy table. See
[Chapter 24](24-the-option1-deep-dive.md).

**Dependency** — an outside package your project relies on, downloaded via
npm. See [Chapter 2](02-installing-tools.md).

**Deployment (of code)** — publishing your app to a hosting service so
it's reachable at a real, public URL. See
[Chapter 28](28-deploying-your-app.md).

**Deployment Frequency** — one of the four DORA metrics: how often
successful deployments happen, expressed as a rate. See
[Chapter 11](11-metrics-explained.md).

**DORA metrics** — four industry-standard metrics (Deployment Frequency,
Change Failure Rate, Lead Time for Changes, Time to Recovery) measuring a
team's software delivery health, from the DevOps Research and Assessment
group. See [Chapter 11](11-metrics-explained.md).

**Dynamic import** — `import(...)` used as a function call rather than a
top-of-file statement, loading a module only when actually needed. See
[Chapter 26](26-exporting-a-pdf-report.md).

**Environment variable** — a named value read by your code at runtime,
kept outside the code itself — the standard way to store secrets. See
[Chapter 15](15-environment-variables-and-secrets.md).

**ESLint** — a linter for JavaScript/TypeScript: a tool that flags
likely-mistaken or inconsistent code patterns without running the code.
See [Chapter 13](13-project-setup.md).

**Execution** (in Harness) — one specific run of a pipeline, with a
status, start time, and end time. See
[Chapter 10](10-harness-and-cd-pipelines.md).

**Fetch layer** — a dedicated module (like `lib/harness.ts`) that is the
only place in your app that knows how to talk to one specific external
API. Piece #3 of the anatomy table. See
[Chapter 17](17-talking-to-the-harness-api.md).

**Generic function/type** (TypeScript) — a function or type written to
work with any type, using a placeholder like `<T>`, rather than one
hardcoded type. See [Chapter 19](19-remaining-api-routes.md).

**Git** — a version control system: software that keeps a full history of
every version of every file in a project. See
[Chapter 3](03-git-and-github-basics.md).

**GitHub** — a hosting service for Git repositories. Not the same thing as
Git itself. See [Chapter 3](03-git-and-github-basics.md).

**Headless component** — a UI component that implements correct
interactive behavior (keyboard navigation, focus management, screen-reader
support) with no visual styling opinions at all. See
[Chapter 9](09-what-is-shadcn-ui.md).

**Hydration / hydration mismatch** — the process of React attaching
interactive behavior onto HTML that was already rendered by the server; a
mismatch occurs when the server's and the client's first render disagree.
See [Chapter 27](27-polish-accessibility-and-bugfixes.md).

**JSX** — HTML-like syntax used inside JavaScript/TypeScript files to
describe UI, understood by React. See [Chapter 6](06-what-is-react.md).

**Lead Time for Changes** — one of the four DORA metrics: how long a
change takes to reach production; implemented in this app as an honestly
-labeled proxy (average/median run duration). See
[Chapter 11](11-metrics-explained.md).

**Lint / linter** — see ESLint.

**localhost** — a hostname that always refers to "this same computer,"
used to reach a locally-running development server. See
[Chapter 13](13-project-setup.md).

**Map** (JavaScript) — a built-in data structure for storing key/value
pairs, useful for grouping records by an arbitrary runtime key. See
[Chapter 17](17-talking-to-the-harness-api.md), [Chapter 22](22-computing-metrics.md).

**Metric** — a single, meaningful number derived from a larger pile of raw
data. See [Chapter 11](11-metrics-explained.md).

**Metrics module** — code (like `lib/dora.ts`) that turns raw records into
summary metrics, exposing one clean function as its entry point. Piece #6
of the anatomy table. See [Chapter 22](22-computing-metrics.md).

**MTTR (Mean Time To Recovery)** — see Time to Recovery.

**Node.js** — a program that runs JavaScript outside of a web browser. See
[Chapter 2](02-installing-tools.md).

**npm (Node Package Manager)** — the tool, bundled with Node.js, used to
download and manage a project's dependencies. See
[Chapter 2](02-installing-tools.md).

**npx** — a tool bundled with npm that downloads and runs a package one
time without permanently installing it — used for one-off commands like
scaffolding a new project. See [Chapter 13](13-project-setup.md).

**Optional chaining (`?.`)** — TypeScript/JavaScript syntax that safely
accesses a property, producing `undefined` instead of crashing if
something earlier in the chain doesn't exist. See
[Chapter 5](05-typescript-crash-course.md), [Chapter 17](17-talking-to-the-harness-api.md).

**Organization (Org)** — in Harness, the top-level grouping that contains
projects. See [Chapter 10](10-harness-and-cd-pipelines.md).

**Pagination** — showing a long list of records a fixed-size "page" at a
time, with controls to move between pages. See
[Chapter 21](21-fetching-and-displaying-executions.md).

**Pipeline** — in Harness, defines the steps to deploy a service. See
[Chapter 10](10-harness-and-cd-pipelines.md).

**Port (networking)** — a numbered "door" a computer can listen for
network connections on; `localhost:3000` means port 3000 on this same
computer. See [Chapter 13](13-project-setup.md).

**Project** (in Harness) — lives inside an org, holds the pipelines,
services, and environments for one team. See
[Chapter 10](10-harness-and-cd-pipelines.md).

**Prop** — data passed into a React component from its parent. See
[Chapter 6](06-what-is-react.md).

**Proxy metric** — a number computed as a stand-in for an ideal
measurement that isn't actually available, used honestly and clearly
labeled as an approximation. See [Chapter 11](11-metrics-explained.md).

**Query parameter** — the part of a URL after a `?`, used to pass named
values to a server, e.g. `?org=my-org-id`. See
[Chapter 19](19-remaining-api-routes.md).

**Rate** — a metric shape: count of events divided by a time window. See
[Chapter 11](11-metrics-explained.md).

**Ratio** — a metric shape: count of a subset divided by count of the
whole. See [Chapter 11](11-metrics-explained.md).

**React** — a JavaScript library for building UI out of reusable
components. See [Chapter 6](06-what-is-react.md).

**Regular expression (regex)** — a compact pattern language for matching
text, e.g. `/ppv3/i`. See [Chapter 17](17-talking-to-the-harness-api.md).

**Sentinel value** — a specific, unlikely-to-collide value used as a
stand-in for a special meaning, flowing through normal code paths like a
real value would. See [Chapter 20](20-building-the-filter-bar.md).

**Server Component** — in Next.js's App Router, the default kind of
component, which runs on the server and can talk directly to
databases/APIs. Contrast with Client Component. See
[Chapter 7](07-what-is-nextjs.md).

**Set** (JavaScript) — a built-in data structure for storing a collection
of unique values, optimized for "is this value a member?" checks. See
[Chapter 17](17-talking-to-the-harness-api.md), [Chapter 21](21-fetching-and-displaying-executions.md).

**shadcn/ui** — a command-line tool that copies editable UI component
source code directly into your project, rather than being installed as an
invisible dependency. See [Chapter 9](09-what-is-shadcn-ui.md).

**State** — data a React component remembers and can change over time,
managed with `useState`. See [Chapter 6](06-what-is-react.md).

**Tailwind CSS** — a utility-first CSS framework, where you compose small
pre-made classes directly in your markup instead of writing custom CSS
classes. See [Chapter 8](08-what-is-tailwind.md).

**Terminal / shell** — a text-based interface for typing commands directly
to your computer. See [Chapter 1](01-terminal-basics.md).

**Time to Recovery (MTTR)** — one of the four DORA metrics: how fast a
team recovers from a failure; implemented in this app as an honestly
-labeled proxy. See [Chapter 11](11-metrics-explained.md).

**Timestamp** — a moment in time represented as a plain number
(milliseconds since a fixed reference point), making time arithmetic
trivial. See [Chapter 16](16-defining-types.md).

**Type** (TypeScript) — a description of the shape of a piece of data. See
[Chapter 5](05-typescript-crash-course.md), [Chapter 16](16-defining-types.md).

**TypeScript** — JavaScript plus a type-checking layer that catches
shape-mismatch mistakes before code ever runs. See
[Chapter 5](05-typescript-crash-course.md).

**Union type** — a TypeScript type that must be one of several specific
values, e.g. `"Success" | "Failed"`. See
[Chapter 5](05-typescript-crash-course.md).

**useEffect** — a React function for running code in response to a
component appearing or its data changing. See
[Chapter 6](06-what-is-react.md).

**useMemo** — a React function that re-runs a calculation only when its
listed dependencies actually change, reusing the previous result
otherwise. See [Chapter 6](06-what-is-react.md), [Chapter 21](21-fetching-and-displaying-executions.md).

**useState** — a React function that gives a component a piece of
remembered, changeable data. See [Chapter 6](06-what-is-react.md).

**"use client"** — a directive at the top of a file marking it as a
Client Component. See [Chapter 7](07-what-is-nextjs.md).

**Version control** — see Git.

**Worker pool / concurrency limiting** — a pattern for running several
async operations at once, capped at a maximum number running
simultaneously, so a shared counter of remaining work gets claimed by
whichever worker is free next. See [Chapter 17](17-talking-to-the-harness-api.md).
