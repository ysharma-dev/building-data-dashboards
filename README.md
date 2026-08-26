# Building Data Dashboards

### A beginner's guide to building real, data-driven dashboard apps — taught by building one from scratch

## What this book actually teaches

This is **not** a book about one specific app. It's a book about a
*skill*: taking data from somewhere (an API, a spreadsheet, a database) and
turning it into a dashboard that helps people understand what's happening
and make decisions.

You'll learn that skill by building one complete, real dashboard from an
empty folder to a finished, deployed app: **Harness Deploy Insights**, a
tool that shows software teams how well they're deploying code, using an
industry-standard framework called [DORA metrics](11-metrics-explained.md).

By the last page, you will have:

- Built a full-stack web application with a modern framework (Next.js),
  a modern language (TypeScript), a design system (Tailwind CSS + shadcn/ui),
  and a charting library (Recharts).
- Learned how to talk to any external API safely, model its data with types,
  and build your own backend layer in front of it.
- Learned the general shape of "compute a metric from a list of records" —
  a pattern that shows up in every dashboard you will ever build, no matter
  the subject.
- Learned how to build filters, comparisons, deep-dive analyses, charts, and
  a PDF export feature.
- Learned to deploy your app so other people can use it.
- Read a final chapter that hands you three fully detailed alternative
  dashboard projects, so you can immediately practice applying everything
  you learned to a completely different subject.

Every chapter follows the same shape: first, the **general skill** — the
part that transfers to any project you'll ever build — then a **worked
example** using the real app, in small, explained pieces of code (never a
giant wall of code to copy-paste), and finally a short reminder of how the
skill generalizes beyond this one app.

## Who this book is for

Total beginners. This book assumes you have never opened a terminal, never
written a line of JavaScript, and don't know what "npm" means. Every term is
explained the first time it's used, and also collected in the
[glossary](appendix-c-glossary.md) at the end.

If you already know some of the early material (e.g., you've used Git
before), skim those chapters — nothing later depends on you having *struggled*
through the basics, just on you understanding them.

## What you'll need

- A computer running macOS, Windows, or Linux.
- About 15-20 hours, spread over however many sessions you like. There's no
  clock — go slowly, re-read sections, and actually type the code rather than
  copying it. Typing it is how it sticks.
- Optionally, a [Harness](https://www.harness.io/) account to pull real
  deployment data. **You do not need one.** [Appendix A](appendix-a-mock-data-fallback.md)
  shows you how to run the entire app with realistic fake data instead, so
  you can follow every chapter either way.

## How the book is organized

**Part 0 — Introduction.** What we're building and why, and a map of the
pieces every dashboard app is made of.

**Part 1 — Foundations (Chapters 1-9).** The tools and languages this project
uses: the terminal, Git, JavaScript, TypeScript, React, Next.js, Tailwind
CSS, and shadcn/ui. Skip chapters you're already confident in.

**Part 2 — Understanding the Domain (Chapters 10-12).** Before writing any
code, we study what we're actually measuring: what Harness is, what DORA
metrics are, and the specific optimization story ("Option 1") this dashboard
was originally built to prove out.

**Part 3 — Building the Project (Chapters 13-28).** Step by step, in the
order a real project gets built: project setup, styling, talking to an API,
building the UI, computing metrics, charting, exporting reports, fixing
real bugs, and deploying.

**Chapter 29 — Building Your Own Dashboard.** The payoff chapter. A
pattern-reference card mapping every piece of this app to the chapter that
taught it, followed by three fully worked-through alternative dashboard
ideas (GitHub activity, weather trends, personal finance) showing exactly
what to reuse and what to rebuild.

**Appendices.** Running the app without a Harness account (mock data),
troubleshooting common beginner errors, and a glossary of every term used.

## Table of contents

- [00 — Introduction](00-introduction.md)

**Part 1 — Foundations**
- [01 — Terminal Basics](01-terminal-basics.md)
- [02 — Installing Your Tools](02-installing-tools.md)
- [03 — Git and GitHub Basics](03-git-and-github-basics.md)
- [04 — JavaScript Crash Course](04-javascript-crash-course.md)
- [05 — TypeScript Crash Course](05-typescript-crash-course.md)
- [06 — What Is React?](06-what-is-react.md)
- [07 — What Is Next.js?](07-what-is-nextjs.md)
- [08 — What Is Tailwind CSS?](08-what-is-tailwind.md)
- [09 — What Is shadcn/ui?](09-what-is-shadcn-ui.md)

**Part 2 — Understanding the Domain**
- [10 — Harness and CD Pipelines](10-harness-and-cd-pipelines.md)
- [11 — Metrics, Explained](11-metrics-explained.md)
- [12 — The "Option 1" Optimization Story](12-the-option1-optimization-story.md)

**Part 3 — Building the Project**
- [13 — Project Setup](13-project-setup.md)
- [14 — Configuring Tailwind and shadcn](14-configuring-tailwind-and-shadcn.md)
- [15 — Environment Variables and Secrets](15-environment-variables-and-secrets.md)
- [16 — Defining Types](16-defining-types.md)
- [17 — Talking to the Harness API](17-talking-to-the-harness-api.md)
- [18 — Your First API Route](18-first-api-route.md)
- [19 — The Remaining API Routes](19-remaining-api-routes.md)
- [20 — Building the Filter Bar](20-building-the-filter-bar.md)
- [21 — Fetching and Displaying Executions](21-fetching-and-displaying-executions.md)
- [22 — Computing Metrics](22-computing-metrics.md)
- [23 — The PPv3 Comparison](23-ppv3-comparison.md)
- [24 — The Option 1 Deep Dive](24-the-option1-deep-dive.md)
- [25 — Charts with Recharts](25-charts-with-recharts.md)
- [26 — Exporting a PDF Report](26-exporting-a-pdf-report.md)
- [27 — Polish, Accessibility, and Bugfixes](27-polish-accessibility-and-bugfixes.md)
- [28 — Deploying Your App](28-deploying-your-app.md)

**The Payoff**
- [29 — Building Your Own Dashboard](29-building-your-own-dashboard.md)

**Appendices**
- [A — Running Without a Harness Account (Mock Data)](appendix-a-mock-data-fallback.md)
- [B — Troubleshooting](appendix-b-troubleshooting.md)
- [C — Glossary](appendix-c-glossary.md)

## A note on the source code

This book explains code in small, understandable fragments — never as a
giant file to copy-paste. You're meant to type it yourself, following the
explanations, and end up with your own working version. The original app
this book is based on is a private project, so instead of linking to it,
every chapter gives you everything you need to write the file yourself from
scratch.

Ready? Start with [Chapter 0 — Introduction](00-introduction.md).
