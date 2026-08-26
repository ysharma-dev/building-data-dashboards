---
layout: chapter
title: "Chapter 9 — What Is shadcn/ui?"
nav_order: 10
permalink: /09-what-is-shadcn-ui/
---

# Chapter 9 — What Is shadcn/ui?

Building every single piece of interface — buttons, dropdowns, dialogs,
tooltips — completely from scratch, styled correctly and behaving properly
for every user (including those using a keyboard instead of a mouse, or a
screen reader), is a huge amount of careful work. Most real projects reach
for a pre-built set of components instead of reinventing all of this. This
book's dashboard uses **shadcn/ui** for exactly that purpose, starting in
Chapter 14. Before you get there, it's worth understanding what shadcn/ui
actually *is* — because it works fundamentally differently from almost
every other component library you may have heard of, and that difference
changes how you'll interact with it for the rest of this book.

## Not a dependency — a copy machine

If you've heard of component libraries like Bootstrap or Material UI, you
might expect shadcn/ui to work the same way: you install it as a
**dependency** (an outside package your project relies on, downloaded via
npm — recall Chapter 2), you import components from it, and the actual
component code lives tucked away inside your `node_modules` folder, mostly
invisible and not meant to be edited.

**shadcn/ui does not work this way, and this is its single most important
idea.** shadcn/ui is a **command-line tool**, not a component library you
depend on. When you run its command to add a component — something like
`npx shadcn@latest add button`, which Chapter 14 will walk through for
real — it does not install a package. Instead, it **copies the actual
source code** of that button component directly into your own project, as
regular files you can open, read, and freely edit, sitting right alongside
the rest of your code.

This has a genuinely different consequence than a normal dependency: **you
own every component, completely.** There's no library maintainer's code
hidden away that you're not supposed to touch — if the button component
needs to look or behave slightly differently for your specific dashboard,
you just open the file and change it, the same way you'd edit any other
file in your project. You're not fighting against, or working around, a
library's assumptions — the "library" is just your own code, from the
moment it's copied in.

The tradeoff is one you'll also want to know about honestly: because
there's no ongoing dependency, shadcn/ui components don't get automatically
updated when a new version is released — if you want an improvement made
to a component after you've copied it in, you'd need to reapply it
yourself. For a huge number of real projects (including this book's), that
tradeoff is well worth it in exchange for full ownership and the ability to
customize freely.

## Base UI: the behavior underneath the styling

shadcn/ui's components don't invent their interactive behavior from
scratch either. Underneath the styling, they're built on top of a separate
library of **headless** components — meaning components that implement
correct *behavior* (keyboard navigation, focus management, ARIA attributes
for screen readers, opening/closing logic) with **no visual styling
opinions at all**. shadcn/ui then layers Tailwind CSS styling (Chapter 8)
on top of that behavior to produce the finished, good-looking component you
actually use.

For this book's project, that underlying headless library is
**[Base UI](https://base-ui.com/)** (the package name is `@base-ui/react`).
Consider a component like a dropdown menu: correctly handling every
keyboard interaction (arrow keys to move between options, `Escape` to
close, focus landing in the right place when it opens and returning to the
right place when it closes), and correctly announcing all of that to
screen-reader software, is substantial, easy-to-get-wrong work — the kind
of work worth not redoing from scratch on every project. Base UI provides
exactly that behavior, unstyled, and shadcn/ui's copied-in component code
wires it up together with Tailwind classes so it also looks right.

## An important heads-up about outside tutorials: Radix UI vs. Base UI

If you go looking at shadcn/ui tutorials, videos, or blog posts online,
there's a very good chance you'll run into a *different* underlying
headless library: **Radix UI**. For most of shadcn/ui's history, Radix UI
was the (and for a long time, the only) headless primitive it was built on,
so the overwhelming majority of existing shadcn/ui material out there —
possibly including material more recent than you'd expect — assumes Radix
UI underneath.

This book's project uses **Base UI**, not Radix UI. Functionally, for
everything you'll do in this book, the difference is invisible — components
look the same, and you use them the same way in your own JSX. But if you
go looking at outside tutorials for extra detail on a specific component,
you may notice small API differences that come directly from this
underlying swap. The one you're most likely to actually run into: many
Radix-based components accept a prop called **`asChild`**, which tells the
component "don't render your own wrapping element — instead, merge your
behavior directly onto the one child element I'm giving you." Base UI's
equivalent mechanism is a prop called **`render`**, which serves a similar
purpose — letting you supply the actual element to render — but with a
different shape (you pass it the element to render into, rather than a
plain `true`/`false`-style flag).

You won't need to use either of these directly for anything covered in this
book. This is mentioned now purely so that if you go looking at an outside
shadcn/ui tutorial later and see `asChild` where this book's actual project
code uses `render`, you'll immediately recognize *why* — it's a Radix UI
vs. Base UI difference, not a mistake in either the tutorial or this book.

## Where this goes next

This chapter deliberately hasn't gone deep on any one specific component —
buttons, cards, dialogs — because that's exactly what
[Chapter 14](14-configuring-tailwind-and-shadcn.md) is for, once your
project actually exists to add them to. What matters for now is the
concept underneath all of it: **shadcn/ui is a tool that copies editable
component source code into your project, and this book's copy of that code
is built on Base UI rather than the more commonly-tutorialized Radix UI.**
Everything else follows from that.

## Checkpoint

- You can explain the key difference between shadcn/ui and a traditional
  component library like Bootstrap: it copies source code into your project
  rather than being installed as an invisible dependency.
- You can explain what a "headless" component is, using keyboard navigation
  and screen-reader support as examples of the behavior it provides without
  any visual styling.
- You know this project uses Base UI, and that many outside tutorials use
  Radix UI instead — and that this is why you might see `asChild` in a
  tutorial where this book's code uses `render`.
- You can explain, in one sentence, why "you own the component code" is a
  meaningful practical benefit, not just a technical detail.

Next: [Chapter 10 — Harness and CD Pipelines](10-harness-and-cd-pipelines.md)
