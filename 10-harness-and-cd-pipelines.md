---
layout: chapter
title: "Chapter 10 — Harness and CD Pipelines"
nav_order: 11
permalink: /10-harness-and-cd-pipelines/
---

# Chapter 10 — Harness and CD Pipelines

## Skill: reading an unfamiliar API's data model before you write a line of code

Here is a scenario that will happen to you many times in your career, no
matter what you end up building: someone hands you access to a system you've
never used before — a piece of software, a web service, a database someone
else designed — and says, "can you build a dashboard on top of this?"

Your instinct, especially early on, might be to start immediately: open your
editor, write a function, try to fetch *something*, see what comes back. Fight
that instinct. The single most useful thing you can do before writing any
code against a new system is to sit down, with no code open at all, and
understand its **data model** — what the actual "things" (called **entities**)
in this system are, how they relate to each other, and how you're allowed to
ask for them.

That's the skill this chapter teaches. We're going to do it for a real system
— a deployment platform called Harness — but pay attention to the *process*,
not just the specific facts about Harness, because you will repeat this exact
process the next time you're handed a system you've never seen before.

Concretely, understanding a data model means answering three questions:

1. **What are the entities?** What are the "nouns" this system deals in? A
   store's system might have Customers, Orders, and Products. A video
   platform might have Channels, Videos, and Comments.
2. **How are they related, and what's the hierarchy?** Does one entity
   *contain* another? Does an Order belong to exactly one Customer, or can it
   belong to several? Is there a broadest, top-level entity that everything
   else lives inside?
3. **How do I ask for them?** Once you know the nouns and their
   relationships, you need to know the actual mechanism for requesting data
   about them — which, for most modern systems, means reading API
   documentation.

We'll walk through all three for Harness. By the end, you'll have a mental
map of exactly what a "deployment" even *is* in this system, which is the
foundation everything else in this book is built on.

## What is Harness, in plain terms?

**Harness** is a platform that automates building, testing, and deploying
software. This category of tool is usually called **CI/CD** — short for
**Continuous Integration** (automatically testing and combining code changes
as different people make them) and **Continuous Delivery/Deployment**
(automatically taking tested code and getting it running somewhere real,
like a live website or an internal server).

This book focuses on the "CD" half. A **CD pipeline** is an automated
sequence of steps that takes a piece of code and deploys it — meaning it
gets that code running on real infrastructure, like a cluster of servers,
so that it actually does something for actual users. Think of a pipeline
as a recipe: "first do this step, then this step, then this step," except
instead of cooking, the steps are things like "package the code," "push it
to the servers," "check that it started up correctly."

Before Harness (or tools like it) existed, teams often deployed code by
hand — someone would SSH into a server, copy files over, and restart a
process, hoping they didn't fat-finger a step. That's slow and error-prone.
A CD pipeline replaces that manual ritual with a repeatable, automatic,
auditable sequence — you press a button (or a pipeline triggers
automatically when code changes), and the same steps run the same way every
time.

Now, why would a team want a *dashboard* about their pipelines at all? Because
once deployments are automated, they also become *measurable*. Every time a
pipeline runs, Harness records what happened: when it started, when it
finished, and whether it succeeded or failed. That history of runs is exactly
the kind of raw material a dashboard is built from — and understanding the
shape of that raw material is this chapter's job.

## Question 1: What are the entities?

Let's figure out the "nouns" in Harness's world. If you spend a few minutes
exploring Harness's own product (or its documentation), you'd notice these
recurring terms, roughly from most-general to most-specific:

- **Account** — your organization's entire presence in Harness. If your
  company uses Harness, you likely have one account for the whole company.
- **Organization** (often shortened to **org**) — a broad grouping inside
  the account. A large company might have one org per business unit — for
  example, "Payments" and "Mobile" might be two separate orgs inside the
  same account.
- **Project** — lives inside an org, and represents one team's area of
  work. A project holds the specific pipelines, services, and environments
  that one team actually uses day to day.
- **Pipeline** — lives inside a project, and defines the actual steps to
  deploy a particular piece of software (a "service," in Harness's
  language) — build it, test it, deploy it to some environment.
- **Execution** — one single run of a pipeline, at one point in time.

That last one, **execution**, is the entity you'll spend the most time with
in this book, so let's be precise about what it actually is. Every time a
pipeline runs — whether triggered by a person clicking a button or
automatically because new code was pushed — Harness creates one execution
record for that run. An execution has, at minimum:

- A unique ID, so you can refer to that specific run later.
- A **status** — typically one of `Success`, `Failed`, `Aborted` (someone
  or something stopped it before it finished), or `Running` (still in
  progress).
- A **start time** — when the run began.
- An **end time** — when the run finished (or, if it's still `Running`,
  there isn't one yet).

If a pipeline is a recipe, an execution is "the specific time you cooked
that recipe last Tuesday at 3pm, and it turned out fine" — as opposed to the
recipe itself, which is just the instructions, unattached to any particular
attempt.

## Question 2: What's the hierarchy?

Now that we have the nouns, notice that they nest inside each other, like
Russian nesting dolls, from broadest to narrowest:

**Account → Organization → Project → Pipeline → Execution**

Read that arrow as "contains." An Account contains one or more
Organizations. Each Organization contains one or more Projects. Each Project
contains one or more Pipelines. And each Pipeline has been run some number
of times, each of those runs being one Execution.

Why does this hierarchy matter to you, specifically, as someone about to
build a dashboard? Because it tells you exactly how your **filters** will
need to work. If you want to let someone narrow down "show me deployments
for just this team, for just this pipeline," you need dropdowns that
respect this same hierarchy: pick an org, and now you can only pick projects
that live inside that org; pick a project, and now you can only pick
pipelines that live inside that project. You'll build exactly this cascading
filter bar later in the book — but notice that the *design* of that filter
bar isn't really a UI decision. It's a direct reflection of the data's
hierarchy that we just wrote down, with no code at all. That's the payoff of
doing this step first: half your later decisions get made for you, for
free, just by understanding the shape of the data.

It's worth pausing on why systems like this tend to be organized this way at
all. A large company might have thousands of pipelines running thousands of
times a day. Without a hierarchy, you'd have one giant flat pile of
pipelines with no way to say "these ones belong to that team." The
Org → Project structure exists specifically so that access, organization,
and — usefully for us — *filtering* can happen at multiple, sensible
levels, rather than one team wading through everyone else's pipelines to
find their own.

## Question 3: How do I ask for this data?

Every one of those entities is retrievable through Harness's **REST API** —
"REST" is a common style of designing web APIs, and an API ("Application
Programming Interface") is, in this context, a set of URLs a program can
request data from, the same way your browser requests a web page, except
the response is structured data instead of a page meant for human eyes.

Harness's current API is often called the **NG API** ("NG" for "Next-Gen,"
distinguishing it from an older, earlier version of Harness's API). It
exposes almost exactly the hierarchy we just mapped out, as separate URLs
you can request data from:

- `/ng/api/organizations` — list the organizations in your account.
- `/ng/api/projects` — list the projects (optionally, within one org).
- `/pipeline/api/pipelines/list` — list the pipelines within a project.
- `/pipeline/api/pipelines/execution/summary` — list executions (runs) for
  a pipeline, or across pipelines.
- `/pipeline/api/pipelines/execution/v2/{id}` — get the full details of
  *one specific* execution, by its ID, including a detailed, nested
  step-by-step graph of everything that happened during that run. You'll
  ask for this with an extra option, `?renderFullBottomGraph=true`, which
  tells Harness "don't just give me the summary — give me every nested
  step, all the way down." You'll see exactly why that matters in Chapter
  12, when we need to look inside a run to find one specific step.

Every one of these requests needs two things to work: an
`accountIdentifier`, passed as part of the URL, telling Harness which
account you're even asking about, and an `x-api-key` header — a header
being an extra piece of information attached to a request, invisible in the
URL itself — which authenticates you, proving you're allowed to see this
account's data at all. You'll learn exactly how headers and authentication
work, and how to keep an API key like this secret, in
[Chapter 15](15-environment-variables-and-secrets.md). For now, just notice
the shape: every single one of these URLs mirrors an entity from our
hierarchy. That's not a coincidence — it's what a *well-designed* API looks
like: URLs that map directly onto the mental model you'd already sketch out
on paper, rather than forcing you to learn some unrelated scheme.

Notice, too, how little of this required writing code. Everything so far
came from reading documentation and thinking. That's deliberate, and it's
the entire point of this chapter: the map of a system should exist in your
head — or on paper — *before* it exists in a text editor. Once you sit down
in [Chapter 17](17-talking-to-the-harness-api.md) to actually write the
functions that call these URLs, you'll already know exactly what you're
building and why, because you did the thinking here first.

## A wrinkle worth knowing about now: pipelines that call other pipelines

There's one more piece of the Harness data model worth flagging here, even
though we won't fully unpack it until Chapter 12. It's a good example of
something you'll run into constantly when working with real systems: the
simple mental model ("a pipeline runs some steps, and finishes") is *mostly*
true, but real systems have wrinkles.

Here's the wrinkle: when you look inside one execution's step-by-step graph,
one of those steps might not be an ordinary step at all — it might have a
`nodeType` (a field telling you what *kind* of step this is) of
`"Pipeline"`. That means this step isn't doing work itself — it's calling
an entirely separate, "child" pipeline to do the real work on its behalf.
That child pipeline has its own execution ID and its own complete
step-by-step graph, which you have to go fetch *separately*, as its own
request.

Why would anyone design it this way? Picture a company with fifty different
teams, each with their own pipeline, but all fifty of those pipelines need
to do the same complicated, careful deployment dance at the end — check some
approvals, verify a cluster is healthy, actually push the new code out. Instead
of copy-pasting those same steps into fifty different pipelines (and then
having to update all fifty if that process ever needs to change), you write
that shared logic *once*, as its own pipeline, and have all fifty
team-specific "parent" pipelines call that one shared "child" pipeline. This
"parent calls child" pattern is common in exactly this kind of shared,
reusable deployment template setup.

For now, just file this away: **an execution's graph can contain a
reference to another, separate execution, which you must fetch on its own.**
We'll return to this in full in Chapter 12, because it turns out to be
essential to a real analysis this book's app performs.

## Try this yourself: the same skill, a different API

The best way to know whether you actually absorbed a skill — rather than
just a fact — is to try applying it somewhere new. So before moving on,
spend a few minutes on this exercise. You don't have to actually write
anything down or open a browser if you don't want to; just think it
through, the same way we did above for Harness.

Imagine you'd been handed access to **GitHub's REST API** instead —
GitHub being the popular website many software teams use to store their
code and track their work. You've probably at least heard of GitHub even if
you've never used it. If you were asked to build a dashboard on top of it,
you'd ask the same three questions:

1. **What are the entities?** GitHub deals in things like Organizations,
   Repositories (a "repo" being one project's stored code), Issues (a
   tracked bug or task), Pull Requests (a proposed code change, waiting for
   review), and Comments (a message left on an issue or pull request).
2. **What's the hierarchy?** Try sketching it yourself, the way we did for
   Harness. You'd land on something like: **Organization → Repository →
   Issue (or Pull Request) → Comment.** An organization contains many
   repositories; each repository contains many issues and pull requests;
   each of those can have many comments.
3. **How would I ask for it?** GitHub, like Harness, publishes REST API
   documentation with a URL for each entity — something like
   `/orgs/{org}/repos` to list an organization's repositories, and
   `/repos/{owner}/{repo}/issues` to list a repository's issues. If you ever
   do build something on top of GitHub's API, you'd read that documentation
   with exactly the same three questions in mind that we just asked about
   Harness.

Notice that you didn't need to already know GitHub's API to answer any of
that — you needed to know the *shape* of the questions to ask. That
transferability is the whole point. The next time you're handed a payments
API, a weather API, a library system's API, or literally anything else
you've never seen before, this is your starting move: find the entities,
find the hierarchy, find how to ask for it. Everything else follows from
there.

## This generalizes to: entity-hierarchy thinking, for any system

The skill from this chapter — identify the entities, map their hierarchy,
then find the mechanism for requesting them — has nothing specifically to
do with Harness, deployments, or even APIs in the narrow sense. It applies
just as well to a spreadsheet someone hands you ("okay, so each row is an
order, orders belong to customers, customers belong to a region..."), a
database schema, or a file format you've never seen before.

Whenever you're about to build *any* dashboard, on *any* subject, this is
step zero, before any code: find the nouns, find how they nest inside each
other, and find how you're allowed to ask for them. Do that thinking on
paper first. It's dramatically cheaper to redraw a hierarchy diagram than
to rewrite code built on a wrong mental model — and once you have that
diagram, decisions later in the project (like how your filters should
cascade) often make themselves.

In the next chapter, we'll take the entity we just spent this whole chapter
understanding — the **execution** — and learn how to turn a big list of them
into four specific, meaningful numbers.

Next: [Chapter 11 — Metrics, Explained](11-metrics-explained.md)
