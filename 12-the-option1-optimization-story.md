# Chapter 12 — The "Option 1" Optimization Story

## Skill: turning a hypothesis into a measurable signal, and hunting down confounds

Imagine someone on your team says: "I think our new caching change is
making the app faster." Or: "I bet our new checkout flow is reducing cart
abandonment." Or, in this book's case: "I bet skipping unnecessary
deployment work is actually saving us real time."

All three of those are **hypotheses** — educated guesses about cause and
effect, stated in plain, informal language. None of them, as stated, are
things you can directly check against data. Before you can investigate any
of them, you have to do a translation step: turn the vague, human-language
claim into a **specific, measurable signal** — an exact thing you can
count, or look up, in the data you actually have.

And once you've done that translation, there's a second, easy-to-skip step
that separates a trustworthy analysis from a misleading one: you have to
ask whether anything *else*, unrelated to the thing you're testing, is
mixed into your measurement. Something like that is called a **confound**
— a factor that corrupts your measurement for reasons that have nothing to
do with what you're actually trying to measure. If you don't find and
remove confounds, you can end up "proving" something is true (or false)
for entirely the wrong reason.

This chapter walks through both steps — hypothesis-to-signal translation,
and confound-hunting — using a real optimization story from the app this
book builds. Pay attention to the *process*; the specific Harness details
are just the vehicle.

## The hypothesis: does skipping unnecessary work actually save time?

Recall from Chapter 10 that a parent pipeline can call a shared "child"
pipeline to do the actual deployment work — a pattern used so that many
teams can reuse the same careful, well-tested deployment logic instead of
each maintaining their own copy.

That shared child pipeline has a smart feature built into it, internally
called **"Option 1."** Here's the idea behind it: redeploying the exact
same configuration you already have running is pointless work — if
nothing changed, there's nothing to gain by going through the full deploy
dance again. So the child pipeline checks, early on, whether anything
actually changed since the last deploy, and if nothing did, it skips the
expensive steps entirely.

The team that built this wanted to know: **does this skip logic actually
save meaningful time?** That's the hypothesis. As stated, it's not
something you can query a database for — "meaningful" isn't a data field,
and neither is "actually save time" on its own. We need to translate it.

## Step 1: translating the hypothesis into a measurable signal

To translate a hypothesis into a signal, ask: "if this hypothesis were
true, what specific, countable thing would I expect to see in the data?"

Let's trace through exactly how Option 1 works, mechanically, because the
signal falls directly out of the mechanism:

1. The child pipeline has an early step called **`Render_Manifest`**. A
   "manifest," here, is a configuration file describing exactly what
   should be running — which container images, how many copies, what
   settings — on a system called Kubernetes (a popular tool for running
   containerized applications across a cluster of servers; you don't need
   to know more about it than that for this book). This step produces an
   output variable — a named piece of data the step hands off to later
   steps — called **`manifestChanged`**, which is simply `true` or
   `false`: did the manifest actually come out different from what's
   currently deployed?
2. If `manifestChanged` is `false`, later steps — **`Check_ArgoCD_Status`**
   (checking whether a deployment tool called ArgoCD confirms everything's
   healthy) and **`Deploy_Application`** (the actual, expensive act of
   pushing the new configuration out) — get **skipped** entirely, since
   there's nothing new to deploy or verify.
3. If `manifestChanged` is `true`, those steps run normally.

So here's our translated, measurable signal: **look at whether
`Deploy_Application`'s status came back as `"Skipped"`.** If it did, the
optimization "fired" for that run. If it ran normally (`Success` or
`Failed`), the optimization didn't fire — either because something
actually changed, or because of some other reason entirely.

Notice what just happened: a vague question ("does this save time?") became
two concrete, countable things:

- **How often did it fire?** Count how many executions have
  `Deploy_Application` marked `"Skipped"`, out of the total. This is a
  **ratio**, the same shape you learned in Chapter 11 — count of a subset
  (skipped runs) ÷ count of the whole (all runs).
- **How much time did it save, when it fired?** Compare the typical
  duration of a *skipped* run against the typical duration of a *normal*
  run. The difference is your estimated time savings per skip.

## Fetching the signal: applying Chapter 10's skill again

Here's where the "parent calls child" wrinkle from Chapter 10 stops being
a foreshadowed curiosity and becomes essential. The steps we just described
— `Render_Manifest`, `Check_ArgoCD_Status`, `Deploy_Application` — don't
live in the parent pipeline's execution at all. They live inside the
**child** pipeline's own, separate execution.

So actually gathering this signal takes a specific sequence, one that
directly reuses what you learned about Harness's data model in Chapter 10:

1. Fetch the parent execution's full step graph (using
   `?renderFullBottomGraph=true`, as you learned in Chapter 10, so nothing
   nested is left out).
2. Look through that graph for a node with `nodeType: "Pipeline"` — the
   signal that this stage is actually a call out to a child pipeline.
3. From that node, extract the child's own execution ID, along with which
   org and project it lives in (remember, a child pipeline might live in a
   different project than its parent).
4. Fetch the *child's* execution graph, separately, using that ID.
5. Inside the child's graph, read `Render_Manifest`'s `manifestChanged`
   output, and `Deploy_Application`'s status.

This is a direct, practical payoff of the entity-and-hierarchy thinking
from Chapter 10. If you hadn't already internalized that an execution's
graph can point to another, separate execution elsewhere, this whole
analysis would be mysterious or even invisible to you — you'd look at the
parent's graph, never find `Render_Manifest` at all (because it isn't
there), and conclude the data simply wasn't available.

## A naming convention, not a real setting: "PPv3"

Before we get to the confound, one more piece worth being upfront about,
because it's a good example of a subtlety in data analysis: sometimes the
label you group things by isn't a formal, guaranteed-correct field — it's
a convention, or a guess, and you should say so.

The app compares "PPv3" pipelines against everything else — PPv3 being a
naming convention some teams use for their pipelines. Here's exactly how
the app decides whether a pipeline counts as "PPv3": it checks whether the
pipeline's **name or identifier contains the text "ppv3"**, ignoring
uppercase/lowercase differences. That's it — it's a simple string match,
not a real, guaranteed field or flag inside Harness's own configuration.

This means the classification is a **heuristic** — an approximate,
practical rule that works well in the common case but isn't logically
guaranteed to be perfectly correct. A pipeline that happens to have "ppv3"
in its name for an unrelated reason would get miscounted; a pipeline that
uses the convention but is named slightly differently would get missed.
The book's app is upfront about this being a simple heuristic rather than
a real Harness setting, and so should you be, in any dashboard you build
that groups data by a naming pattern rather than an explicit, structured
field. Whenever a grouping is "does the name contain this substring"
rather than "does this record have this explicit property," it's worth a
sentence in your own documentation (or your UI) saying so.

## Step 2: hunting for confounds

Now for the part of this chapter that matters most. Suppose you naively
computed "average time saved when Option 1 fires" by just comparing the
total duration of skipped runs against the total duration of normal runs,
start to finish. Would that number be trustworthy?

It would not — and here's why. Remember from Chapter 10 that a full
deployment involves more than just the mechanical deploy steps. The real
child pipeline also contains two entirely separate stages:

- **"Approve Deploy"** — a gate that gets stuck waiting for a **human** to
  approve a Jira ticket (Jira being a common tool for tracking work
  items and approvals) before the pipeline is allowed to continue.
- **"Validate Clusters"** — a Harness **manual-approval gate**, which
  again waits for a human to click "approve" before proceeding.

Both of these stages can take an unpredictable, sometimes very long amount
of time — not because of anything related to the deploy mechanism itself,
but because they're waiting on a **person**. Maybe that person is in a
meeting. Maybe they're at lunch. Maybe it's a ticket that took two days to
get prioritized. None of that has anything to do with whether Option 1's
skip logic is working well or saving time.

If you included those wait times inside your "total duration" numbers,
here's the problem: a run that happened to skip the deploy work (because
nothing changed) but got stuck for 45 minutes waiting on a human to click
"approve" would look, in your naive total-duration number, like a *slow*
run — even though the optimization itself worked perfectly, and the actual
deploy mechanism finished in seconds. Averaged across enough runs, human
approval delays — pure noise, unrelated to the thing you're actually
studying — would swamp the real signal you're trying to measure, in either
direction, essentially at random depending on who happened to be at lunch
that day. That's a confound: a factor mixed into your number that has
nothing to do with what the number claims to measure.

**How the real app handles this.** Rather than either (a) silently
including these wait times, which would corrupt the skip-rate and
time-saved numbers, or (b) silently deleting/ignoring them, which would
throw away real information a viewer might want, the app does a third
thing: it measures the approval wait times **separately**, keeps them in
their own clearly named fields — `approveDeployDurationMs` and
`validateClustersDurationMs`, each with its own start and end timestamps —
and **excludes them entirely** from the skip-rate and time-saved
calculations. Then, rather than hiding that this exclusion happened, it
shows those excluded wait times in their own clearly labeled panel in the
dashboard, explicitly described as "excluded from the numbers above."

This is the general move worth remembering: when you find a confound, you
have three options — include it (corrupting your measurement), silently
discard it (losing real information, and hiding a decision you made from
whoever reads your dashboard), or **isolate and label it**, which is what
the app does. That third option is almost always right. It keeps your
headline number honest and clean, while still being transparent that
something real happened and got set aside, and exactly what it was.

## A non-technical example, to make the pattern land

Let's step completely outside of software to make sure this pattern is
clear, because it applies everywhere.

Suppose a retail store hypothesizes: "rearranging our shelves into this
new layout will increase sales." To test it, they translate the vague
hypothesis into a measurable signal: total dollars in sales, this month,
compared to last month, in the store that changed its layout.

Sales go up 20%. Success? Not necessarily — you first have to ask: **was
anything else different this month that could explain the increase, for
reasons that have nothing to do with the shelf layout?** Suppose it turns
out the store also ran a 15%-off storewide discount during that exact
same month, as an unrelated, independent decision. That discount is a
confound — it would plausibly increase sales on its own, regardless of
where anything sits on a shelf, and it's now tangled up inside the same
number you're trying to use to judge the shelf layout.

The fix mirrors exactly what the app does: don't just report "sales are up
20%, so the new layout works." Isolate the confound — perhaps by comparing
against a similar store that got the new layout but *not* the discount, or
by separately tracking how much of the sales increase is attributable to
the discounted items specifically — and report the layout's *effect*
separately from the discount's effect, rather than one blended, ambiguous
number.

Here's a second example, just to reinforce the pattern with a different
flavor of confound: suppose a school wants to know whether a new tutoring
program improved students' test scores. If the tutoring program happened
to launch at the same time the school also adopted an easier textbook, any
score improvement is now confounded between two independent causes. The
fix is the same shape again: identify the second factor, and find a way to
isolate its effect — perhaps by comparing tutored versus non-tutored
students who both used the new textbook — rather than crediting the whole
improvement to tutoring by default.

## The general checklist

Whenever you're investigating whether some change or optimization
"worked," run through this sequence:

1. **State the hypothesis in plain language.** "I think X causes Y."
2. **Translate it into a specific, measurable signal.** What exact field,
   count, or comparison in your actual data would tell you whether X
   happened, and whether Y followed?
3. **Ask what else might be mixed into that signal.** Is there some other
   factor, changing at the same time or present in the same numbers, that
   could produce the same effect (or mask it) for reasons unrelated to
   your actual hypothesis?
4. **Isolate and label any confound you find**, rather than silently
   including it (which corrupts your measurement) or silently discarding
   it (which hides a real decision from whoever reads your results).

## This generalizes to: hypothesis, signal, confound — for any investigation

Take away the specific words "Option 1," "manifest," and "Harness," and
what's left is a completely general recipe for investigating *any* claim
about *any* data: turn the vague hypothesis into an exact, countable
signal; then actively hunt for anything else mixed into that signal that
has nothing to do with what you're trying to measure; then isolate and
clearly label whatever you find, rather than quietly folding it in or
quietly throwing it away.

This is the difference between a dashboard that looks impressive and a
dashboard people actually trust. Anyone can compute a number and put it in
a big colorful box. What makes a number *trustworthy* is being able to
answer, clearly, "what exactly does this measure, and what have you made
sure is **not** hiding inside it?" That question — and the habit of
answering it before you ship a metric, not after someone catches a
mistake — is the most valuable thing this chapter has to offer, and it
will serve you on every dashboard you ever build, whether the subject is
software deployments, retail sales, school test scores, or anything else.

With the domain now fully understood — what Harness is, what the four DORA
metrics actually measure (and where two of them are honest proxies), and
how the Option 1 story turns a hypothesis into a clean, confound-free
measurement — you're ready to start actually building. Part 3 begins with
getting your project set up from scratch.

Next: [Chapter 13 — Project Setup](13-project-setup.md)
