---
layout: chapter
title: "Chapter 11 — Metrics, Explained"
nav_order: 12
permalink: /11-metrics-explained/
---

# Chapter 11 — Metrics, Explained

## Skill: turning a list of records into a handful of meaningful numbers

In the last chapter, you learned what an **execution** is: one run of a
deployment pipeline, with a status, a start time, and an end time. By
itself, a single execution doesn't tell you much. But a *list* of hundreds
of them, collected over months, is a goldmine — if you know how to
squeeze meaning out of it.

That squeezing process is called computing a **metric**: a single,
meaningful number derived from a larger pile of raw data. This is arguably
the single most important skill in this entire book, because it's the
actual "brain" of any dashboard. Filters, charts, and tables are all just
different ways of *presenting* metrics — but the metrics themselves are
where the real thinking happens.

Nearly every metric you'll ever compute, across any dashboard, on any
subject, is built from one of three basic shapes:

1. **A rate over time** — how often something happens, expressed per some
   unit of time. "Count of events, divided by a time window." Deploys per
   day. Support tickets closed per week. Steps walked per hour.
2. **A ratio between a subset and the whole** — what fraction of your data
   falls into some category you care about. "Count of X, divided by count
   of everything." Percentage of orders that got returned. Percentage of
   emails that got opened. Percentage of deployments that failed.
3. **A proxy** — a number you compute because the number you *actually*
   want isn't measurable with the data you have, so you compute something
   related and *honestly label it as an approximation*, rather than
   pretending it's the real thing.

That third one is easy to get wrong, and easy to do dishonestly by accident
— so we'll spend real time on it. Getting proxies right is a mark of a
trustworthy dashboard; getting them wrong (or hiding the fact that a number
is a proxy at all) is how dashboards quietly mislead the people relying on
them.

We're going to learn all three shapes through a real, industry-standard
set of four metrics called **DORA metrics** — but keep an eye on which
shape each one is, because that's the part you'll reuse forever.

## What are DORA metrics?

**DORA** stands for **DevOps Research and Assessment** — a research group
(originally independent, now part of Google) that spent years studying
what separates software teams that ship reliably and quickly from teams
that don't. Their research converged on four specific numbers that,
together, give a surprisingly complete picture of a team's deployment
health. They're widely used across the software industry today, which is
exactly why the app in this book computes them.

The four metrics are:

1. **Deployment Frequency** — how often you ship.
2. **Change Failure Rate** — how often a deployment breaks something.
3. **Lead Time for Changes** — how long a change takes to go live.
4. **Time to Recovery** — how fast you recover when something breaks.

Notice already: two of the words in there are "Frequency" and "Rate" —
your first hint that we're dealing with the rate and ratio shapes from
above. Let's take them one at a time, with the *exact* formula the real
app uses for each — verified against its actual source code, not
simplified or approximated by us.

## Metric 1: Deployment Frequency (a RATE)

**What it answers:** "How often does this team actually ship code?"

**The formula:**

> (count of executions with status `Success`) ÷ (span in days between the
> oldest execution's start time and the newest execution's start time, in
> the data you're currently looking at — with a minimum window of 1 day,
> so you never accidentally divide by zero)

That last clause matters and is easy to overlook: if every execution in
your current view happened to start on the exact same day, the "span"
between oldest and newest would be zero days — and dividing by zero is
either an error or a nonsensical "infinity" result, depending on your
programming language. So the app enforces a floor: even if the real span
is less than a day, treat it as 1 day. You'll hit this "floor to avoid
divide-by-zero" trick constantly once you start computing rates yourself —
it's not a Harness-specific hack, it's a universal defensive habit.

**Worked example.** Suppose you're looking at a pipeline with 8 successful
executions, and the earliest one started 4 days ago while the most recent
started today. The span is 4 days.

| Quantity | Value |
|---|---|
| Successful executions | 8 |
| Span (days) | 4 |
| Deployment Frequency | 8 ÷ 4 = **2 per day** |

**Why is this hard to read as a raw number?** "2" isn't self-explanatory —
2 *what*? Per day? Per week? A dashboard's job is to make numbers
readable at a glance, so the app converts this raw rate into whichever
unit reads most naturally: if the rate is at least 1 per day, show it as
"X per day"; otherwise, if it's at least 1 per week, show "X per week";
otherwise, show "X per month." A team deploying twice a day should see
"2 per day," not "0.014 per hour" or some other technically-correct but
unreadable unit. This is a small but real lesson about dashboards in
general: computing the right number is only half the job. Presenting it in
whatever unit a human will actually parse instantly is the other half.

**Which direction is "good"?** Higher is better here. Industry research
(the DORA research this whole framework is named after) found that "elite"
performing teams deploy **on-demand**, often multiple times a day — the
opposite of the old world where a "deployment" was a nerve-wracking event
that happened once a quarter. Frequent deployment, done well, is a sign
of a mature, low-friction, well-automated process — small changes going
out constantly are far less risky than huge changes going out rarely.

## Metric 2: Change Failure Rate (a RATIO)

**What it answers:** "Out of everything we shipped, what fraction of it
broke something?"

**The formula:**

> (count of executions with status `Failed` OR `Aborted`) ÷ (total count of
> all executions) × 100 — expressed as a percentage

Note the "OR" — this metric doesn't just count outright `Failed`
executions. An `Aborted` execution (one that someone or something stopped
before it finished, rather than letting it run to a natural failure) still
represents a deployment attempt that did not successfully complete, so it
counts against you here too. Deciding *which statuses count as "bad"* is
itself a real decision a metric designer has to make deliberately — it's
not automatic, and getting it wrong (say, forgetting to include `Aborted`)
would quietly understate how often things go wrong.

**Worked example.** Say a project ran 20 total executions this month: 15
succeeded, 3 failed outright, and 2 were aborted partway through.

| Quantity | Value |
|---|---|
| Total executions | 20 |
| Failed | 3 |
| Aborted | 2 |
| Failed + Aborted | 5 |
| Change Failure Rate | 5 ÷ 20 × 100 = **25%** |

**Which direction is "good"?** Lower is better — you want the fraction of
bad deployments to be as close to zero as possible. DORA's research puts
"elite" teams under 15%. Our worked example above, at 25%, would not
qualify as elite — which is exactly the kind of at-a-glance judgment a
dashboard should let someone make instantly, without them needing to know
DORA's benchmarks by heart. That's part of why the real app shows small
contextual labels next to each metric card, rather than a bare number.

## Metric 3: Lead Time for Changes (a PROXY)

Here's where we hit our third shape, and where honesty starts to matter a
lot.

**What DORA actually means by this metric:** the time from when a
developer **commits** code (saves a finished change into the project's
shared history, using a tool called Git — you'll learn Git in
[Chapter 3](03-git-and-github-basics.md)) to the moment that exact code is
**running in production** — meaning live, actually serving real users.
That's the *true*, complete definition: idea-to-customer time, start to
finish.

**Why the app can't actually compute that.** To measure "time from commit
to production," you'd need to cross-reference two different systems:
Harness (which knows when deployments ran) and a source-control system
like Git (which knows when a specific code commit was made) — and you'd
need to reliably link a specific commit to the specific deployment that
first shipped it. This app only has access to Harness's own execution
data. It has no visibility into commit timestamps at all. So the *true*
Lead Time for Changes is simply not something this app's data can produce.

**What the app computes instead — and why this matters.** Rather than
silently pretending to answer the real question, the app computes
something related but narrower: the average and the median total run
duration (time from an execution's start to its finish) of **successful**
executions only. In plain words: "how long does a deploy itself take to
run, once it starts?" — not "how long does an idea take to reach
customers?" Those are genuinely different questions. A team could have a
blazing-fast pipeline (start to finish in 10 minutes) while the underlying
code sat in a queue for two weeks before anyone even started that
pipeline — a scenario where the *proxy* number would look great while the
*true* metric would look terrible.

This is exactly what a **proxy metric** is: a stand-in you compute because
the ideal, truest measurement isn't available to you, used honestly, and
clearly labeled as an approximation rather than presented as the real
thing. The real app labels this directly in its user interface with a
small "proxy" badge next to the metric, plus a plain-language explanation
of exactly what it's actually measuring versus what DORA's original
definition calls for. This book models that same honesty: whenever you
build a metric that isn't quite measuring what its name implies, say so,
visibly, right next to the number. The alternative — silently showing a
number under DORA's official name, while quietly measuring something
narrower — isn't a shortcut, it's a way of misleading whoever reads your
dashboard, even if you didn't mean to.

**Which direction is "good"?** Lower is better — shorter run durations
mean less time between starting a deploy and it finishing successfully.

## Metric 4: Time to Recovery (a PROXY, with real subtleties)

**What DORA actually means by this metric:** often written as **MTTR**
(Mean Time To Recovery), this is the time from when an incident or outage
*starts* to when it's *resolved*. Critically, a real-world incident might
get resolved by something that has nothing to do with a deployment tool at
all — an engineer manually restarting a server, hand-editing a
configuration file, or flipping a setting in some other system entirely.

**Why the app can't actually compute that, either.** Just like Lead Time,
this app can only see activity that happened through Harness. It has no
way of knowing about a manual fix that happened outside of it. So instead,
it computes a proxy: **for every execution with status `Failed` or
`Aborted`, how long until the next `Success` on that same pipeline?** The
reasoning: if a deployment failed and the *next* successful deployment
(via the same tool) probably represents the fix going out. It's an
approximation of real-world recovery time, using only the signal
available — which is a perfectly reasonable thing to do, as long as you
label it as what it is, exactly like Lead Time above.

Now here's where this metric gets genuinely instructive, because getting
it *correct* — not just "roughly right," but actually correct — required
handling two real subtleties. Both are worth understanding deeply, because
you will run into cousins of both problems constantly when working with
any real, timestamped data.

### Subtlety A: you must group by pipeline first

Imagine two completely unrelated pipelines — say, one for a company's
mobile app, one for its billing system — both owned by different teams,
deploying on completely independent schedules. If you failed to keep them
separate, and just looked at *all* executions across the whole company
sorted by time, you might see the mobile app's pipeline fail at 2:00pm,
and the billing pipeline happen to succeed at 2:15pm — and wrongly
conclude "recovery took 15 minutes," when in reality those two events have
absolutely nothing to do with each other. They're on independent
timelines; one's success says nothing about the other's failure being
fixed.

The fix is straightforward once you see the problem: **group executions by
pipeline first**, and only ever compare a failure to a later success
*within that same group*. This is a broadly useful habit: whenever you're
comparing timestamps across records, ask whether those records actually
belong to the same independent "lane," or whether you're accidentally
mixing unrelated timelines together.

### Subtlety B: overlapping executions can create nonsensical negative time

This one is subtler, and it's a real bug that was found and fixed during
this project's actual development — a genuinely useful cautionary tale.

Here's the scenario. Imagine one pipeline has two executions running
around the same time (this can absolutely happen — for instance, someone
manually re-triggers a deploy while a previous, slower one is still
finishing):

- **Execution A** starts at 2:00pm, is slow, and doesn't finish (aborting)
  until 2:30pm.
- **Execution B** starts later, at 2:10pm, but is fast, and succeeds by
  2:15pm.

If you naively look for "the next success, by start time, after the
failure," you'd find Execution B — it started at 2:10pm, which is after
Execution A's start time of 2:00pm. You'd then compute a "recovery time" as
the gap between them — but B's start time (2:10pm) is actually *before* A
even finished (2:30pm)! You'd end up computing a **negative recovery
time**: the "fix" appears to have happened before the problem was even
done failing. That's nonsensical, and it's a direct symptom of trusting
raw start-time ordering when executions can overlap in time.

**The fix:** don't compare against the next success's *start* time by
sequence order. Instead, require that the next success's start time be
**greater than or equal to the failing execution's own end time.** In our
example, Execution A doesn't end until 2:30pm, so Execution B (which
started at 2:10pm, before A even ended) doesn't qualify as "the recovery"
at all — the app has to keep looking for the next success that started at
or after 2:30pm.

The general lesson here, worth internalizing well beyond this one metric:
**when your data can contain overlapping or concurrent records, you cannot
trust "the next one" by simple ordering alone.** You have to think
carefully about what "next" actually should mean — often, as here, that
means comparing an end time against a start time, not just comparing two
start times to each other. This is a mistake that's easy to make once
(the original app did) and easy to avoid forever once you've seen it
explained clearly — which is exactly why it's worth a whole section here
rather than a passing mention.

**Which direction is "good"?** Lower is better — a shorter gap between a
failure and its eventual fix means less time spent in a broken state.

## Seeing the three shapes side by side

| Metric | Shape | What it divides |
|---|---|---|
| Deployment Frequency | Rate over time | Successful executions ÷ days spanned |
| Change Failure Rate | Ratio (subset ÷ whole) | Failed-or-aborted executions ÷ all executions |
| Lead Time for Changes | Proxy | Average/median run duration, standing in for commit-to-production time |
| Time to Recovery | Proxy | Time from failure to next real success on the same pipeline, standing in for true incident-resolution time |

Notice that "rate" and "ratio" are really just two flavors of the same
underlying idea — division — applied to slightly different things: a rate
divides a count by an amount of *time*, while a ratio divides a count by
another, larger *count*. Once you notice that, most metrics you'll ever
need to invent start to feel a lot less mysterious. They're almost always
some count, divided by something.

## This generalizes to: rates, ratios, and honest proxies, anywhere

Strip away the words "deployment," "pipeline," and "Harness" from
everything above, and you're left with three shapes that show up in
literally every domain that produces event data over time:

- **A rate** — count of events ÷ a time window. Support tickets closed
  per day. New sign-ups per week. Miles run per month.
- **A ratio** — count of a subset ÷ count of the whole. Percentage of
  orders that get returned. Percentage of emails that bounce.
  Percentage of job applicants who get an interview.
- **A proxy** — used whenever the number you actually want isn't
  measurable with the data you have. If a store wants to measure "customer
  satisfaction" but only has return-rate data, return rate becomes a
  proxy for satisfaction — useful, but not the same thing, and worth
  saying so out loud rather than quietly relabeling "return rate" as
  "satisfaction."

Whenever you sit down to design a metric for a dashboard on any subject,
run through these same three shapes and ask which one fits what you're
trying to measure — and if it's a proxy, ask what real subtlety might be
hiding underneath it, the way overlapping executions hid underneath Time
to Recovery. That habit of asking "is this actually measuring what I
think it's measuring, and is there a messy edge case in the data that
could quietly corrupt it?" is worth far more than knowing DORA's specific
formulas. The next chapter puts that exact habit to work on a much
trickier question: not just "is this metric well-defined," but "is this
metric accidentally counting things that shouldn't count at all?"

Next: [Chapter 12 — The "Option 1" Optimization Story](12-the-option1-optimization-story.md)
