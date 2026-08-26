# Chapter 1 — Terminal Basics

Every chapter after this one is going to ask you to type a command into a
black (or white) window with blinking text, instead of clicking an icon.
That window is called a **terminal**, and it's the single most-used tool in
this entire book — you'll use it to install software, start your app, check
what changed in your code, and much more. If you've never opened one before,
this chapter is where that changes. Nothing here requires you to know what
programming is yet. We're just learning to talk to your computer in a new
way.

## A graphical file browser vs. a terminal

You already know how to browse files on your computer: you open Finder (on
a Mac) or File Explorer (on Windows), you see folders and icons, and you
double-click things. That's a **graphical user interface**, or GUI
(pronounced "gooey") — you interact with pictures (icons, windows, buttons)
using a mouse.

A **terminal** (also called a **shell**, a **command line**, or a
**console** — you'll see all these words used almost interchangeably) is a
different way of doing the exact same kind of thing: looking at files,
moving them, creating them, deleting them — except instead of clicking
icons, you type words. There's no picture of a folder. There's just text: you
type a command, press a key, and text appears telling you what happened.

This might sound like a downgrade. It isn't, for a few reasons that will
matter a lot later in this book:

- **It's precise.** "Create a folder named `harness-deploy-insights` inside
  the folder named `projects`" is one exact typed instruction, with no room
  for misclicking.
- **It's automatable.** You can save a sequence of typed commands and run
  them all again later, or share them with someone else so they get the
  exact same result you did. You can't really do that with mouse clicks.
- **It's how developer tools actually work.** Installing Node.js (Chapter
  2), saving your work with Git (Chapter 3), and starting your dashboard app
  (Chapter 13) all happen by typing commands. There isn't a graphical
  alternative for most of it — the terminal *is* the interface.

You're not replacing Finder or File Explorer. You'll still use those. You're
adding a second way to do the same kinds of things, and for the specific
job of building software, it's the way that's actually used.

## Opening a terminal

### On macOS

macOS comes with a built-in terminal application called **Terminal**. To
open it:

1. Press `Cmd + Space` to open Spotlight search (a search box that appears
   in the middle of your screen).
2. Type `Terminal`.
3. Press `Enter` when "Terminal" appears as the top result.

A window will open — usually with a white or black background and some text
already in it, ending in a blinking cursor. That's your terminal, ready for
input.

You can also find it manually: open Finder, go to
**Applications → Utilities → Terminal**.

### On Windows

Windows comes with a few different terminal options. This book recommends
**Windows Terminal** running **PowerShell**, which is the modern standard
and comes pre-installed on current versions of Windows 10 and 11.

1. Click the Start menu (or press the Windows key).
2. Type `Terminal`.
3. Click "Terminal" (or "Windows Terminal") when it appears.

If you don't have Windows Terminal, typing `PowerShell` into the Start menu
search and opening "Windows PowerShell" works too — the commands in this
book behave the same way in both.

> **A note on commands across operating systems.** Most of the commands in
> this chapter work identically on macOS and Linux, because both are built
> on a shell language called **bash** (or a close relative of it, like
> **zsh**, which is the current macOS default). Windows PowerShell uses
> mostly the same core commands but has a few of its own names for things.
> This chapter will point out the differences as they come up.

## The prompt

Once your terminal is open, you'll see a line of text sitting at the bottom
(or wherever your cursor is), usually ending in a symbol like `$`, `%`, or
`>`, followed by a blinking cursor. This is called the **prompt** — it's the
terminal telling you "I'm ready, type something."

A macOS prompt might look like this:

```bash
yourname@MacBook-Pro ~ %
```

A Windows PowerShell prompt might look like this:

```bash
PS C:\Users\yourname>
```

The exact text varies by computer (it often includes your username and
where you currently are), but the idea is the same everywhere: it's waiting
for you.

## Typing a command and pressing Enter

To use the terminal, you type a word (or a few words) and press the `Enter`
key (sometimes labeled `Return` on Mac keyboards). Pressing Enter tells the
terminal "I'm done typing, run this now." The terminal then does whatever
you asked, prints any resulting text below your command, and shows you a
fresh prompt when it's finished — ready for your next command.

Nothing happens until you press Enter. You can type, delete, retype, and fix
typos all you want on that one line before you commit to running it.

## The "working directory"

Here's a concept that trips up a lot of beginners, so let's slow down: at
every moment, your terminal is "standing" in one specific folder on your
computer. That folder is called your **working directory** (a **directory**
is just another word for a **folder** — the terminal world tends to say
"directory," so we'll use both interchangeably from here on).

Think of it like this: if a graphical file browser is a window that always
shows you exactly one folder's contents, your terminal is a window that
doesn't show you anything visually, but is still always "inside" exactly
one folder. Every command you type that deals with files — creating one,
listing what's there, deleting one — happens relative to that folder unless
you say otherwise.

When you open a fresh terminal, it usually starts you in your **home
directory** — the main personal folder for your user account (things like
`Documents`, `Downloads`, and `Desktop` typically live inside it).

## The essential commands

Below are the handful of commands you'll use constantly, for the rest of
this book and beyond. Try each one as you read it — the best way to learn a
terminal is to type into one.

### `pwd` — "print working directory"

This command asks the terminal: "Where am I right now?" It prints the full
path of your current working directory.

```bash
pwd
```

Example output:

```bash
/Users/yourname
```

That text — `/Users/yourname` — is a **path**: a precise description of
where a folder lives, written as a chain of folder names separated by `/`
(on macOS/Linux) or `\` (on Windows). Reading it left to right: there's a
folder called `Users`, inside it a folder called `yourname` — and you are
currently standing inside that one.

> On Windows PowerShell, the equivalent command is `pwd` too (PowerShell
> is friendly about accepting a lot of familiar Unix-style command names) —
> it will print something like `C:\Users\yourname`.

### `ls` (macOS/Linux) or `dir` (Windows) — list contents

This command asks: "What's inside the folder I'm currently in?"

```bash
ls
```

On Windows PowerShell, the equivalent is:

```bash
dir
```

(PowerShell also accepts `ls` as a shortcut for the same thing, so either
works there.)

The output is a list of file and folder names sitting inside your current
working directory — the same things you'd see if you opened that folder in
Finder or File Explorer, just written as plain text instead of icons.

### `cd` — "change directory"

This command moves your working directory somewhere else — the terminal
equivalent of double-clicking a folder to go inside it.

```bash
cd Documents
```

This moves you *into* a folder named `Documents` that lives inside your
current folder. Run `pwd` afterward and you'll see your location has
changed.

A few special shortcuts are worth knowing:

- `cd ..` — move **up** one level, into the parent folder (the folder that
  contains the one you're in).
- `cd ~` — jump straight back to your home directory, from anywhere.
- `cd` by itself (with nothing after it) also jumps to your home directory
  on macOS/Linux.

You can also jump multiple levels at once by chaining names with `/`:

```bash
cd Documents/projects
```

This moves you into `projects`, which is inside `Documents`, which is
inside wherever you started — all in a single command.

### `mkdir` — "make directory"

This creates a brand-new, empty folder inside your current working
directory.

```bash
mkdir practice-folder
```

After running this, `ls` (or `dir`) will show a new folder named
`practice-folder` sitting alongside whatever was already there.

### `touch` (macOS/Linux) or a redirect (Windows) — create an empty file

On macOS/Linux, `touch` creates a new, empty file with the name you give it:

```bash
touch notes.txt
```

Windows PowerShell doesn't have `touch` built in by default. The
equivalent way to create an empty file there is:

```bash
"" > notes.txt
```

This uses `>`, which means "send the output of whatever's on the left into
the file named on the right." Sending an empty bit of text (`""`) into a
file that doesn't exist yet creates it, empty. You'll see this `>` pattern
again — it's a general way of saving text output into a file instead of
just printing it to the screen.

### `rm` — remove

This deletes a file. Be careful with this one: unlike dragging a file to
the Trash or Recycle Bin, `rm` typically deletes it **immediately and
permanently** — there's no undo, no Trash folder to recover it from.

```bash
rm notes.txt
```

To delete a folder (and everything inside it), macOS/Linux requires an
extra flag, `-r` (meaning "recursive" — apply this action to the folder and
everything inside it, including folders inside folders):

```bash
rm -r practice-folder
```

On Windows PowerShell, the equivalent commands are `del` for a file and
`rmdir` (with a `-Recurse` flag) for a folder:

```bash
del notes.txt
rmdir -Recurse practice-folder
```

> **Why this matters later:** you won't use `rm` heavily in this book, but
> you'll see it in troubleshooting instructions (Appendix B) — for example,
> "delete the `node_modules` folder and reinstall." Knowing what deleting
> from the terminal actually does (no undo!) matters before you run it.

## Reading command output

When you run a command, one of a few things happens:

- **It prints information and returns you to the prompt**, like `pwd` or
  `ls`. Read the printed text — it's the answer to what you asked.
- **It runs silently and just returns you to the prompt**, like a
  successful `mkdir` or `cd`. In terminal culture, "no news is good news" —
  many commands only print something when there's a problem.
- **It prints an error.** If you mistype a command name, reference a folder
  that doesn't exist, or otherwise ask for something impossible, the
  terminal will print a message explaining what went wrong, often starting
  with something like `command not found` or `No such file or directory`.
  Don't panic when you see red text or the word "error" — read the message,
  it's usually telling you exactly what to fix (a typo, a wrong folder,
  etc.).

A tip for later chapters: when a command seems to "hang" — the cursor just
sits there blinking with no new prompt appearing — it usually means the
program is still running (for example, Chapter 13 will have you start a
dashboard app that keeps running in the terminal on purpose, so it can keep
serving your web page). That's normal, not broken.

## Practice exercise

Try this short sequence now, in order, in your own terminal:

1. Run `pwd` to see where you currently are.
2. Run `cd Desktop` (or `cd Documents`, if you'd rather not clutter your
   Desktop) to move into a folder you can find later. If it says no such
   directory, run `ls`/`dir` first to see what folders actually exist where
   you are, and pick one of those instead.
3. Run `mkdir terminal-practice` to create a new folder.
4. Run `cd terminal-practice` to move inside it.
5. Run `pwd` again — confirm the path now ends in `terminal-practice`.
6. Run `ls` (or `dir`) — it should print nothing, since the folder is empty.
7. Create a file inside it: `touch hello.txt` (macOS/Linux) or
   `"" > hello.txt` (Windows).
8. Run `ls` (or `dir`) again — you should now see `hello.txt` listed.

If you got through all eight steps and saw what was described, you've just
navigated a filesystem, created a folder, moved into it, and created a file
— entirely by typing. That's the whole foundation the rest of this book
builds on.

## Checkpoint

- You can open a terminal (Terminal.app on macOS, Windows Terminal/PowerShell
  on Windows).
- Running `pwd` prints a path, and you understand it describes your current
  working directory.
- You created a folder with `mkdir`, moved into it with `cd`, and confirmed
  it was empty with `ls`/`dir`.
- You understand that pressing `Enter` is what actually runs a typed
  command.

Next: [Chapter 2 — Installing Your Tools](02-installing-tools.md)
