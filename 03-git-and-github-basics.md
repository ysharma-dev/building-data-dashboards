---
layout: chapter
title: "Chapter 3 — Git and GitHub Basics"
nav_order: 4
permalink: /03-git-and-github-basics/
---

# Chapter 3 — Git and GitHub Basics

Somewhere around Chapter 13, you'll start writing real files for your
dashboard app, and you'll want a safety net: a way to save snapshots of
your work as you go, see exactly what you changed at each step, and undo a
mistake without losing everything else. That safety net is called Git, and
this chapter teaches you just enough of it to use throughout the rest of
the book. This is not a complete Git tutorial — entire books exist on Git
alone — it's the practical minimum you need to follow along.

## What is version control, and why does it matter?

Imagine you're writing an important document, and you save it as
`report.docx`. Partway through, you make some changes you're not sure
about. If they don't work out, your only options are "remember what it
used to say" or "undo a bunch of times and hope you land on the right
spot." If you close the file, even `Ctrl+Z` can't save you anymore.

**Version control** is a system that solves this properly: it keeps a
complete history of every saved snapshot of your project, forever, so you
can look back at any past snapshot, compare two snapshots to see exactly
what changed between them, or restore an old snapshot if something breaks.
Think of it as **undo history for your whole project** — not just your last
few keystrokes, but every meaningful checkpoint you've ever chosen to save,
going back to the very first one.

**Git** is by far the most widely used version control system. It runs
entirely on your own computer (you installed it in
[Chapter 2](02-installing-tools.md)) and tracks changes to the files inside
one specific folder — your project folder.

This matters immensely for the app you're about to build: as you go through
Part 3 of this book, you'll be able to save a working checkpoint after each
chapter, so if something breaks later, you can always look back (or roll
back) to the last point where things worked.

## `git init` — starting to track a folder

Git doesn't track every folder on your computer automatically — you have to
tell it, once, "start watching this specific folder." You do that by
running one command inside that folder:

```bash
git init
```

This creates a hidden folder named `.git` inside your project folder (files
and folders starting with a dot are hidden from normal view, but they're
still there). That hidden folder is where Git stores your entire history —
you'll never open or edit it directly, but it's what makes everything else
in this chapter work.

You only run `git init` once per project, right at the start.

## `git status` — what's changed?

This is the command you'll run more than any other Git command in this
book. It asks: "Compared to my last saved snapshot, what's different right
now?"

```bash
git status
```

Its output tells you things like which files you've created or edited that
haven't been saved into Git's history yet (Git calls these **untracked** or
**modified** files), and which files are ready to be included in your next
snapshot. Running `git status` costs nothing and changes nothing — it's
purely informational, so get in the habit of running it often, especially
right before and after making a snapshot.

## `git add` — choosing what goes into the next snapshot

Git doesn't automatically include every changed file in your next
snapshot. Instead, you explicitly choose which changes to include, using
`git add`. This is called **staging** a file — you're putting it into a
holding area, ready to be included in the next snapshot.

To stage one specific file:

```bash
git add app/page.tsx
```

To stage every changed file in the project at once:

```bash
git add .
```

(Remember from Chapter 1: a single dot means "this folder, right here.")

Being able to choose exactly what goes into each snapshot is genuinely
useful — it means a single snapshot can represent one coherent change
("added the filter bar"), rather than a messy mix of unrelated edits. For
this book, though, `git add .` followed by a commit after each chapter is
a perfectly reasonable habit to build.

## `git commit` — saving the snapshot

Once you've staged the changes you want, `git commit` actually saves them
as a permanent snapshot in your project's history:

```bash
git commit -m "Add the filter bar component"
```

The `-m` flag stands for "message," and the text after it is your **commit
message** — a short, human-readable description of what changed and, more
usefully, *why*. A commit message like `"Add the filter bar component"` is
far more useful to your future self than `"changes"` or `"fix stuff"` —
imagine scrolling through months of history later, trying to find the
snapshot where you first added filters. A clear, specific message makes
that search take seconds instead of minutes.

Every commit is permanent and keeps a full copy of what your project looked
like at that moment. You can make as many commits as you like — there's no
limit, and they cost nothing.

A typical rhythm, which you'll repeat constantly in Part 3 of this book,
looks like this:

```bash
git status
git add .
git commit -m "Describe what you just did"
```

## What is GitHub? (And how is it different from Git?)

This trips up almost every beginner at least once, so let's be precise:
**Git** and **GitHub** are not the same thing.

- **Git** is the version control *tool* itself — the program that tracks
  history, running entirely on your own computer. Nothing about Git
  requires the internet.
- **GitHub** is a *website and hosting service* that stores copies of Git
  project histories online, so you can back them up, share them with other
  people, and collaborate. GitHub is one of several such services (GitLab
  and Bitbucket are other well-known ones) — none of them are Git itself,
  they're all places that *host* Git histories.

Think of it like the difference between a word processor and cloud storage:
the word processor (Git) is what lets you write and save versions of a
document at all; the cloud storage service (GitHub) is a separate place you
can additionally upload it to, so it's backed up and others can see it.

This book's Part 3 chapters focus on Git running locally on your machine,
since that's all you strictly need to follow along and build the app. If
you'd like to also back your project up to GitHub or share it, that
involves creating a free account at [github.com](https://github.com),
creating an empty repository there, and running a couple of additional Git
commands (`git remote add` and `git push`) to send your local history up to
it — but that's optional for following this book, so we won't dwell on it
further here.

## What is `.gitignore`?

Not every file in your project folder should be tracked by Git. A
`.gitignore` file (note the leading dot — it's a hidden file, same idea as
the hidden `.git` folder from earlier) lists file and folder names that Git
should simply ignore — never track, never include in a snapshot, never
show in `git status`.

Two categories of files almost always belong in `.gitignore`:

- **Huge, regeneratable folders.** Chapter 13 will have you run
  `npm install`, which downloads hundreds of packages into a folder called
  `node_modules`. That folder can be enormous (often hundreds of megabytes)
  and can always be perfectly recreated by running `npm install` again — so
  there's no value in tracking it, and real cost (a slow, bloated history)
  in doing so.
- **Secrets.** This is the important one, and it's worth its own section.

## Why `.env.local` must never be committed

Later in this book — [Chapter 15](15-environment-variables-and-secrets.md)
— you'll create a file named `.env.local` that holds sensitive values your
app needs, like a private API key that lets your app talk to a service on
your behalf (think of it like a password). We're not setting that file up
yet; this is just a warning to plant early, before you're holding anything
sensitive.

A `.gitignore` file must list `.env.local` (Chapter 13's project setup will
make sure of this automatically) so that Git never tracks it, meaning it
can never accidentally end up in a commit.

Here's why this matters so much more than it might sound like it does:
if a secret value ever gets committed into Git's history, deleting the file
afterward **does not remove it**. Git's entire purpose is to remember every
past snapshot — so if a secret appears in even one old commit, it's still
sitting in your project's history, retrievable by anyone who can see that
history, even after you delete the file and make a brand-new commit. Fully
scrubbing a secret from Git history after the fact is a genuinely difficult,
error-prone process (it involves rewriting history itself), and if that
history was ever pushed to a public GitHub repository, search engines,
bots, and automated scanners can find and exploit leaked secrets within
minutes of them appearing — long before you'd notice and try to remove
them. Real API keys committed to public repositories get used by attackers
routinely; this isn't a hypothetical risk.

The fix is entirely preventative: never let the secret-holding file get
tracked by Git in the first place, by keeping it listed in `.gitignore`
from before the file even exists. We'll come back to this concretely in
Chapter 15, but the habit starts now: **treat `.gitignore` as something you
check, not something you assume is correct.**

## Checkpoint

- You can explain, in your own words, the difference between Git and
  GitHub.
- You understand that `git status` is a safe, read-only command you can run
  anytime to see what's changed.
- You can describe the three-step rhythm: `git add`, then `git commit -m
  "..."`, and why the commit message matters.
- You understand why a file like `.env.local` must be listed in
  `.gitignore` *before* it ever contains a real secret — not after.

Next: [Chapter 4 — JavaScript Crash Course](04-javascript-crash-course.md)
