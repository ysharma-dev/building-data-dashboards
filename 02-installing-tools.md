---
layout: chapter
title: "Chapter 2 — Installing Your Tools"
nav_order: 3
permalink: /02-installing-tools/
---

# Chapter 2 — Installing Your Tools

Before you can build the dashboard app this book walks through, your
computer needs a handful of programs installed: something to actually run
your code, something to save your work's history, and something comfortable
to write code in. This chapter installs all three. By the end, your
terminal (from [Chapter 1](01-terminal-basics.md)) will be able to run
JavaScript code, track file changes with Git, and you'll have a proper code
editor open and ready.

## What is Node.js?

Normally, JavaScript — the programming language you'll start learning in
[Chapter 4](04-javascript-crash-course.md) — only runs inside a web browser
(Chrome, Safari, Firefox, and so on). A browser has a built-in JavaScript
engine that reads JavaScript code on a web page and runs it.

**Node.js** (usually just called "Node") takes that same kind of
JavaScript engine and lets it run *outside* a browser, directly on your
computer, from the terminal. This matters because a huge amount of what
you'll do in this book — starting your dashboard app, running tools that
check your code for mistakes, downloading other people's pre-written code
to use in your project — is done by running JavaScript programs from the
terminal, with no browser involved at all. Node.js is what makes that
possible.

## What is npm?

When you install Node.js, a second tool called **npm** ("Node Package
Manager") comes bundled with it automatically — you don't install it
separately.

Here's the problem npm solves: almost no real project is written entirely
from scratch. Developers constantly reuse small, pre-written pieces of code
that other people have published — for formatting dates, making network
requests, drawing charts, and thousands of other things. These reusable
pieces are called **packages** (or **libraries** — you'll see both words).

npm is the tool that downloads packages you need, keeps track of exactly
which ones (and which versions) your project depends on, and lets you run
scripts that a project defines (like "start the app" or "check the code for
mistakes"). Later in this book (Chapter 13), you'll run a command like
`npm install` and watch it download every package the dashboard app needs,
and `npm run dev` to start the app running. Both of those are npm at work.

## Installing Node.js and npm

Go to [nodejs.org](https://nodejs.org). You'll see a download button for
the **LTS** version — LTS stands for "Long-Term Support," meaning it's the
stable, recommended version rather than the newest experimental one.
Download and run the installer for your operating system (macOS or
Windows), and click through it using the default options — there's nothing
in this book that requires custom install settings.

Once the installer finishes, close and reopen your terminal (installers
often need a fresh terminal window to be recognized), then verify the
installation worked by asking Node.js to report its own version:

```bash
node --version
```

You should see output like this (the exact numbers may differ slightly
depending on when you install):

```bash
v22.14.0
```

Now check npm the same way:

```bash
npm --version
```

```bash
10.9.2
```

If both commands print a version number instead of an error like `command
not found`, the installation worked. If you get an error, close your
terminal completely, reopen it, and try again — a fresh terminal window is
often all that's needed for your computer to recognize newly installed
programs.

> **A note on `--version`.** The two dashes followed by a word, like
> `--version`, are called a **flag** (sometimes "option"). Flags modify
> what a command does. `node --version` doesn't run a JavaScript program —
> the `--version` flag tells Node to just report which version of itself is
> installed and stop. You'll see many more flags throughout this book.

## Optional: nvm, a Node version manager

You don't need this to follow the book, but it's worth knowing it exists:
**nvm** ("Node Version Manager") is a tool that lets you install and switch
between *multiple* versions of Node.js on the same computer. Different
projects sometimes need different Node versions, and reinstalling Node
every time you switch projects would be painful — nvm lets you keep several
versions around and pick which one is active with a single command.

If you only ever plan to work on one project at a time (very common for
beginners), the plain installer above is all you need. If later on you find
yourself juggling multiple projects with conflicting Node version
requirements, search for "nvm" (there are separate versions for
macOS/Linux and Windows) and come back to this idea then.

## Installing Git

**Git** is the tool [Chapter 3](03-git-and-github-basics.md) is entirely
about — it tracks the history of changes to your project's files, so you can
always see what changed, when, and undo mistakes. You need it installed now
so it's ready when Chapter 3 explains how to use it.

**On macOS:** Git is often already installed. Check by running:

```bash
git --version
```

If you see a version number (like `git version 2.43.0`), you're done. If
instead your Mac prompts you to install "Command Line Developer Tools,"
click "Install" and wait for it to finish — that installs Git (and a few
other tools) for you.

**On Windows:** Download the installer from
[git-scm.com](https://git-scm.com). Run it and click through the default
options — the defaults are sensible for this book's purposes. Once it's
done, open a fresh terminal and confirm:

```bash
git --version
```

```bash
git version 2.43.0
```

Any recent version is fine — the exact number doesn't matter for this book.

## Picking a code editor

So far, everything has been about running programs. Now you need somewhere
to *write* them.

You could technically write code in a plain text editor — Notepad on
Windows, TextEdit on macOS. But a **code editor** is a text editor built
specifically for writing code, with features a plain text editor doesn't
have:

- **Syntax highlighting** — different parts of your code (keywords, text,
  numbers) are shown in different colors, making it far easier to read and
  spot mistakes.
- **Autocomplete** — as you type, it suggests what you probably mean to
  type next, based on what's available in your code.
- **Error and warning squiggles** — it can often tell you about a mistake
  (like a typo in a variable name) before you even run your code, by
  underlining it, similar to a spellchecker.
- **Integrated terminal** — most code editors include a terminal window
  built right in, so you don't need to juggle separate windows.
- **Extensions** — small add-ons that add support for specific languages,
  tools, and workflows (you'll install a couple of these for this project).

This book recommends **Visual Studio Code** (almost always shortened to
**VS Code**) — it's free, works identically on macOS and Windows, and is
the most widely used code editor for web development, which means
tutorials, extensions, and troubleshooting help online are easy to find. (It
is unrelated to Visual Studio, an older, much heavier Microsoft product —
the similar name causes confusion sometimes.)

To install it:

1. Go to [code.visualstudio.com](https://code.visualstudio.com).
2. Download the version for your operating system.
3. Run the installer using the default options.
4. Open VS Code once to confirm it launches.

### Opening a project folder in VS Code

Throughout this book, you'll open your project folder in VS Code and use
its built-in terminal to run commands, rather than switching back and forth
to a separate terminal application. To open a folder:

- **From VS Code:** File → Open Folder... and select your project folder.
- **From the terminal:** navigate to your project folder using `cd` (as you
  learned in Chapter 1), then run `code .` — the single dot means "this
  folder, the one I'm currently standing in." This opens VS Code directly on
  that folder. (If `code .` doesn't work the first time, open VS Code
  manually once, then look up "install code command in PATH" for your
  operating system — this is a one-time setup step some installations need.)

To open VS Code's built-in terminal once a folder is open, use the menu
**Terminal → New Terminal**, or the keyboard shortcut `` Ctrl + ` `` (the
backtick key, usually above Tab) on both macOS and Windows.

From this point in the book onward, "open a terminal" and "run this in VS
Code's terminal" mean the same thing — use whichever is more convenient for
you.

## Checkpoint

- Running `node --version` prints a version number, not an error.
- Running `npm --version` prints a version number, not an error.
- Running `git --version` prints a version number, not an error.
- VS Code is installed, and you can open a folder in it and open its
  built-in terminal with `` Ctrl + ` ``.

Next: [Chapter 3 — Git and GitHub Basics](03-git-and-github-basics.md)
