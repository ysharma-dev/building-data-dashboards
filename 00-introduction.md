# Chapter 0 — Introduction

## The goal isn't the app. It's what building the app teaches you.

Imagine you finish this book and, a month later, someone at work says: "It'd
be great if we had a dashboard showing how our support tickets are trending."
Or: "Can we see which of our products are returned the most?" Or: "I want a
personal page that shows how my running pace has changed this year."

None of those are about software deployments. None of them involve Harness,
or DORA metrics, or anything specific to this book's example. But if you've
worked through this book properly, you'll recognize all three as *the same
shape of problem* you already solved — just with different data.

That's the actual goal here: not "you now know how to build this one app,"
but "you now know how to build dashboards, period." We're going to prove
that by building one specific, real, useful dashboard together — in full,
with nothing skipped or hand-waved — and by the end, naming exactly which
parts of it would change if the subject changed, and which parts wouldn't.

## What we're building

The app is called **Harness Deploy Insights**. Here's the story behind it,
because understanding *why* it exists will make every later decision make
sense.

Software teams that use a tool called [Harness](https://www.harness.io/) to
automate their deployments (more on what that means in
[Chapter 10](10-harness-and-cd-pipelines.md)) wanted to answer a question:
"Is our deployment process actually fast and reliable?" That's a vague
question, so the industry has settled on four specific, measurable numbers
that answer it — called **DORA metrics** (you'll learn exactly what they are
in [Chapter 11](11-metrics-explained.md)):

1. **Deployment Frequency** — how often you ship.
2. **Change Failure Rate** — how often a deployment breaks something.
3. **Lead Time for Changes** — how long a change takes to go live.
4. **Time to Recovery** — how fast you recover when something breaks.

Separately, one specific team had built an optimization (internally called
"Option 1") that skips unnecessary deployment work when nothing actually
changed, and they wanted proof that it was actually saving time. That
required a much more detailed, "deep dive" style of analysis — not just four
summary numbers, but a drill-down into individual deployment runs.

So the finished app does two things:

- Shows the four DORA metrics for any pipeline (or every pipeline in a
  project at once), with filters for organization, project, pipeline, and
  status, and a side-by-side comparison view.
- Offers a detailed "deep dive" that measures exactly how often the
  optimization fired, how much time it saved, and shows that trend over
  time — with a button to export the whole thing as a shareable PDF.

If none of those words mean anything to you yet, that's fine — that's what
the next twelve chapters are for. What matters right now is the *shape* of
what we're building, because that shape is universal.

## The anatomy of a dashboard app

Every real dashboard app — this one, or the "support ticket trends" one your
coworker asks for next year — is built from the same handful of pieces,
assembled the same way. Here they are. Keep this table in mind; we'll refer
back to it constantly, and [Chapter 29](29-building-your-own-dashboard.md)
will return to this exact list at the very end of the book.

| # | Piece | What it does | In plain terms |
|---|-------|--------------|-----------------|
| 1 | **Data source** | Where the raw information lives | An external API, a file, a database — something that has the facts |
| 2 | **Typed model** | A description of the shape of that data | "A deployment has an ID, a status, a start time, and an end time" |
| 3 | **Fetch layer** | Code that knows how to ask the data source for data | Functions like `listOrgs()` or `listExecutions()` |
| 4 | **API routes** | Your own small backend, sitting between your UI and the data source | Keeps secrets safe, hides the outside API's quirks |
| 5 | **Filter UI** | Lets a person narrow down what they're looking at | Dropdowns, date pickers, search boxes |
| 6 | **Metrics module** | Turns a list of raw records into a handful of meaningful numbers | "73% of deploys were on time" |
| 7 | **Comparison view** | Puts two subsets of the data side by side | "Team A vs. Team B," "this month vs. last month" |
| 8 | **Deep-dive view** | A more detailed, focused analysis of one specific question | "Exactly how much did this one change help?" |
| 9 | **Charts** | Visual representations of the data over time or by category | Line charts, bar charts, scatter plots |
| 10 | **Export** | Turning the on-screen view into something shareable | A PDF, a CSV download, a shareable link |
| 11 | **Deployment** | Making the app available to other people, not just your own laptop | Publishing it to a URL |

Every single chapter in Part 3 of this book builds exactly one of these
pieces (or a supporting piece underneath it), in this order. When you get to
Chapter 29 and start planning your own dashboard for a completely different
subject, you'll go through this exact list again, piece by piece — deciding
what changes and what you can reuse untouched.

## What you'll actually see when it's done

Since this is a written book rather than a video, here's a description of
the finished app so you have a picture in your head before we start.

At the top of the page, there's a header with the app's name and an "Export
as PDF" button. Below that, a row of dropdowns: **Org**, **Project**,
**Pipeline**, and **Status** — pick your way down through them (each one's
options depend on what you picked in the one before it) to choose exactly
which deployments you want to look at.

Once you've picked something, four cards appear showing the DORA metrics —
each with a small "?" icon you can hover over for a plain-language
explanation of exactly what's being calculated and how. Below that are two
collapsible sections: one comparing "PPv3" pipelines (a specific naming
convention used by the team that requested this dashboard) against
everything else, and one that's the detailed "Option 1" deep-dive — complete
with a bar chart of skip rates, a scatter plot of deployment durations over
time, a sortable table you can expand row-by-row, and a callout panel
showing exactly how much time was excluded for approval waits (so the
numbers aren't secretly padded by how long a human took to click "approve").

Everything responds instantly to the filters at the top, works in both light
and dark mode, and can be exported as a clean, paginated PDF at any time.

## A promise about how this book is written

Three rules we'll follow in every single chapter from here on:

1. **No unexplained jargon.** The first time a term appears, it's defined in
   plain language, right there — not just in the glossary. If you ever hit a
   term that feels unexplained, that's a bug in the book; check the
   [glossary](appendix-c-glossary.md), and know that a real book would have
   fixed it.
2. **No giant code dumps.** You'll see small, complete, working fragments —
   a function at a time, a component at a time — each one explained line by
   line where it matters. You are meant to type these yourself. Typing code
   (rather than copy-pasting) is one of the fastest ways to actually learn
   to read it.
3. **Every "Building" chapter names the general skill first.** Before you
   see a single line of Harness-specific code, you'll see a short "**Skill:**"
   line that tells you what you're really learning — the part that will
   still be useful when you're building something that has nothing to do
   with software deployments.

## What you need before starting

Nothing except a computer and time. If you've never used a terminal, never
written code, or don't know what "Git" is — perfect, that's exactly who
Chapters 1-3 are for. If you already know some of that, feel free to skim
ahead; nothing later requires you to have struggled through material you
already know.

One optional thing: a free or trial [Harness](https://www.harness.io/)
account, so you can pull real data instead of practice data. You don't need
this to follow the book — [Appendix A](appendix-a-mock-data-fallback.md)
shows you how to run everything with realistic fake data instead, and every
chapter works either way.

Let's get your computer set up. Turn to
[Chapter 1 — Terminal Basics](01-terminal-basics.md).
