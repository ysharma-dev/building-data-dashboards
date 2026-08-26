# Chapter 27 — Polish, Accessibility, and Bugfixes

**Skill:** a debugging mindset, and a short list of recurring bug *classes*
that show up again and again in dashboard apps specifically — not because
this app was unusually buggy, but because these particular mistakes are
easy to make in any project with async data, timestamps, and server-
rendered UI. Recognizing the *shape* of a bug, rather than memorizing this
one specific fix, is what actually transfers to your next project.

Every bug in this chapter is real — something that genuinely happened
during this app's development, found, diagnosed, and fixed. Treat this
chapter less like a list of "gotchas to memorize" and more like a set of
detective stories: each one shows a symptom, walks through how the cause
was actually found, and ends with the fix and the general lesson.

## Bug class 1: double-filtering the same data twice

**The symptom:** after adding a rollout cutoff date
([Chapter 24](24-the-option1-deep-dive.md)), the "Execution duration over
time" chart showed *only* orange ("after rollout") dots — no blue ones at
all, even though the underlying data clearly had executions from before
the cutoff too.

**How it was actually found:** rather than guessing, the fix started by
re-reading exactly which array was being passed into the chart component,
and tracing it backward to where it came from. It turned out
`executionsForSkipAnalysis` — already filtered down to only
post-cutoff executions, deliberately, for the skip-rate table — was being
reused as the input to the duration chart. But the duration chart *itself*
tries to split whatever it receives into "before" and "after" groups. Fed
an array that had *already* been filtered to post-cutoff-only, there was
nothing left to put in the "before" bucket — hence, only orange dots.

**The fix:** keep two clearly-named arrays with two distinct purposes —
one scoped for one specific analysis, one left unfiltered for anything
that needs to do its own before/after split — rather than reusing a
filtered array in a second place that assumes it's getting the full set.
[Chapter 24](24-the-option1-deep-dive.md) covers the actual fix in detail;
what's worth internalizing here is the *general* shape of the mistake:

**The general lesson:** whenever a value has already been filtered,
sorted, or transformed for one specific purpose, and you're about to hand
it to a *second* piece of code, stop and ask "does this second piece
expect the *raw* input, or does it expect something already pre-processed
the way the first piece needed?" If those two expectations don't match,
you'll get exactly this kind of bug — not a crash, just silently wrong or
incomplete-looking output, which is often *harder* to notice than an
outright error.

## Bug class 2: trusting simple ordering with concurrent/overlapping data

**The symptom:** the Time to Recovery metric ([Chapter 11](11-metrics-explained.md))
occasionally computed a *negative* recovery time — a physically
nonsensical result, since time can't run backward.

**How it was actually found:** by printing out the actual failure/success
pairs the calculation was matching, for real data, and looking at their
raw timestamps side by side. This revealed the pattern:
a slow-aborting execution didn't finish (record its end time) until well
after a separate, faster execution on the same pipeline had already
started *and succeeded*. The calculation was matching "the next success by
start-time order," which is not the same question as "the next success
that happened after this failure was actually over."

**The fix:** require the candidate "next success" to have a start time at
or after the *failing* execution's own end time — not just later in
simple chronological start-time order. [Chapter 11](11-metrics-explained.md)
covers this fix's exact mechanics.

**The general lesson:** whenever the records you're working with can
overlap in time — two things happening concurrently, rather than strictly
one after another — you cannot safely assume "sorted by start time" tells
you the true sequence of *completion*. Any time you're pairing an event
with "the next thing that happened after it," ask explicitly: after it
*started*, or after it *finished*? Those can give different, sometimes
contradictory answers whenever overlap is possible — and in real-world
timestamped data (deployments, support tickets, shifts, anything with a
duration rather than an instant), overlap is almost always possible.

## Bug class 3: server/client rendering mismatches (hydration errors)

**The symptom:** the browser's developer console showed a "hydration
mismatch" warning/error, naming a `disabled` attribute that differed
between what the server initially rendered and what the client (browser)
rendered immediately afterward.

**Some background first, since this bug class needs it:** recall from
[Chapter 7](07-what-is-nextjs.md) that Next.js can render a page's *initial*
HTML on the server, then hand it to the browser, which then "hydrates" it
— attaching React's interactive behavior on top of that already-rendered
HTML, rather than throwing it away and rendering from scratch. For this to
work correctly, React expects the HTML the *server* produced and the HTML
the *client* would produce, given the exact same inputs, to match. When
they don't, React logs a warning (and, in some cases, visibly "flickers"
or briefly shows incorrect content) while it silently discards the
server's version and re-renders using the client's.

**Why this app hit it:** a value like `hasSelection` (whether the reader
has picked an org, project, and pipeline) depends on state that starts
out identical on the server and on the very first client render — but a
few controls' `disabled` state depended on *data that's only known after
the browser has started running* (like whether a background fetch has
resolved yet), which is, by definition, something the server can't know
in advance when it renders the initial HTML.

**The fix, in two parts:**

1. Where a value's boolean-ness could be computed unambiguously the same
   way on both server and client, make sure it's computed as an explicit,
   real `Boolean(...)`, not something that could technically evaluate
   differently in edge cases (like relying on truthy/falsy propagation
   through several `&&` operators, where a partially-loaded intermediate
   value might not evaluate exactly the same both times).
2. For the small number of controls where the disabled state *genuinely and
   correctly* depends on client-only information (like "has this fetch
   resolved yet," which the server can never know), add
   `suppressHydrationWarning` — a real React prop that tells React "I know
   this one specific attribute is expected to differ between server and
   client for this element; don't warn about it." This isn't a way to
   silence real bugs — it's an explicit, narrow acknowledgment for the one
   attribute where a genuine, harmless, one-time mismatch is expected by
   design.

**The general lesson:** hydration warnings are worth actually investigating
rather than reflexively suppressing. Sometimes (as in part 1 above) they
point to a real, fixable ambiguity in how a value gets computed. Only once
you've confirmed a mismatch is genuinely expected and harmless — because
the value in question truly can't be known until the client runs — is
`suppressHydrationWarning` the right, narrow tool, applied to exactly the
attribute that needs it, not sprinkled broadly across a component "just in
case."

## Bug class 4: an accessibility gap hiding in plain sight

**The symptom:** long pipeline, org, and project names in dropdown menus
were visually truncated with an ellipsis (`…`) to avoid overflowing their
container — a reasonable space-saving choice — but there was no way to see
the *full* name at all, for any name too long to fit.

**How it was actually found:** by trying the app as a real user would,
with realistically long names, and noticing there was genuinely no way to
read the rest of a truncated name — not a "nice to have" gap, but a real
usability problem for anyone with slightly longer-than-average names in
their Harness account.

**The fix:** a small, reusable `TruncatedText` component (built once,
reused everywhere a name might overflow) that wraps the truncated text in
a [Chapter 9](09-what-is-shadcn-ui.md)-style tooltip, showing the *full*,
untruncated text on hover or keyboard focus:

```tsx
"use client";

import { Tooltip, TooltipContent, TooltipTrigger } from "@/components/ui/tooltip";
import { cn } from "@/lib/utils";

export function TruncatedText({ text, className }: { text: string; className?: string }) {
  return (
    <Tooltip>
      <TooltipTrigger
        render={<span className={cn("block truncate text-left", className)} tabIndex={0} />}
      >
        {text}
      </TooltipTrigger>
      <TooltipContent side="top" align="start" className="max-w-md break-words">
        {text}
      </TooltipContent>
    </Tooltip>
  );
}
```

Two details worth calling out: `truncate` (a Tailwind utility, recall
[Chapter 8](08-what-is-tailwind.md)) is what actually clips the text with
an ellipsis at the CSS level — the tooltip is what *recovers* the
information that clipping visually hides. And `tabIndex={0}` matters for
accessibility specifically: without it, someone navigating by keyboard
(rather than a mouse) would have no way to focus this element at all, and
therefore no way to trigger the tooltip via focus — recall from
[Chapter 9](09-what-is-shadcn-ui.md) that correct keyboard behavior is
exactly the kind of thing a good headless component library, and careful
use of it, is meant to guarantee.

**The general lesson:** any time you truncate text for visual space,
you've created an accessibility and usability debt — some information is
now hidden. Pay that debt immediately, in the same component, rather than
treating "show the full text somehow" as a follow-up task that might
never actually get done. A `title` attribute (a simpler, native HTML
tooltip) is a lighter-weight alternative worth knowing about too, though
it doesn't work well for keyboard-only navigation the way a proper focus-
triggered tooltip component does.

## A short catalog, for pattern-matching next time

| Bug class | What to watch for | The general question to ask |
|---|---|---|
| Double-filtering | Data reused in two places with different filtering expectations | "Is this array raw, or already filtered for a different purpose?" |
| Trusting simple ordering | Records with a start *and* end time, potentially overlapping | "Am I comparing start-to-start, or does completion order actually matter here?" |
| Hydration mismatches | A value that depends on client-only information (a fetch, `window`, browser storage) | "Could the server and the client compute this differently, even once?" |
| Hidden information | Any truncated, collapsed, or abbreviated display | "Is there still a way to get the full information, for every kind of user?" |

## Checkpoint

- [ ] You can describe, from memory, the general *shape* of each of the
      four bug classes above — not the specific fix, the pattern.
- [ ] You can explain why "sorted by start time" and "sorted by
      completion order" are not always the same thing.
- [ ] You can explain when `suppressHydrationWarning` is an appropriate
      tool to reach for, and when it would be masking a real bug instead.
- [ ] `TruncatedText` (or your own equivalent) is used everywhere a name
      might overflow its container in your app.

**This generalizes to:** every one of these four bug classes will find you
again, in a completely different project, wearing a different disguise.
The specific symptom will look different — maybe it's a chart showing only
one color for a different reason, or a negative number somewhere that
should never be negative, or a console warning you're tempted to just
silence. What transfers is the *questions* this chapter taught you to ask
when you see those symptoms, not the specific line of code that happened
to fix them here.

Next: [Chapter 28 — Deploying Your App](28-deploying-your-app.md)
