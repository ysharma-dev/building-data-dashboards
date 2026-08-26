---
layout: chapter
title: "Appendix B — Troubleshooting"
nav_order: 32
permalink: /appendix-b-troubleshooting/
---

# Appendix B — Troubleshooting

Common problems beginners hit while working through this book, organized
by symptom. If something's broken and you don't see it here, the most
useful next step is almost always to re-read the exact error message
slowly, word by word — error messages are far more informative than they
first appear, and half of debugging is just actually reading them
carefully rather than skimming past them.

## "command not found: node" / "command not found: npm"

Node.js isn't installed, or your terminal can't find it. Revisit
[Chapter 2](02-installing-tools.md) and confirm `node --version` and
`npm --version` both print a version number. If you just installed
Node.js and this still happens, try closing and reopening your terminal —
some installers only update your terminal's available commands for
*new* terminal windows, not ones already open.

## "Port 3000 is already in use"

Something else on your computer is already using port 3000 — commonly, an
earlier `npm run dev` you forgot was still running in another terminal
window. Find and close that other terminal, or let Next.js pick a
different port automatically (it usually offers to, printing a message
like "Port 3000 is in use, trying 3001 instead" — just use whichever port
it actually starts on).

## "Module not found: Can't resolve '...'"

You're importing something — a file, or a package — that doesn't exist
where your code says it does. Two common causes:

- **A typo in an import path.** Double-check the exact spelling and
  capitalization against the actual file name — `@/lib/Types` and
  `@/lib/types` are different paths on some operating systems (Linux and
  macOS's default settings are case-sensitive for file paths; Windows
  usually isn't, which can hide this exact mistake until you deploy).
- **A package that hasn't been installed yet.** If the error names an
  npm package (like `recharts` or `jspdf`) rather than one of your own
  files, check that you actually ran the `npm install` command for it —
  revisit the relevant chapter (Chapter 25 for Recharts, Chapter 26 for
  the PDF libraries) to confirm.

## TypeScript errors you don't understand

Run `npx tsc --noEmit` directly in your terminal (recall
[Chapter 16](16-defining-types.md)) — this often shows a clearer, more
complete error message than what your editor displays inline. Read the
*first* error in the list first; TypeScript sometimes reports several
follow-on errors that are really just consequences of one earlier mistake,
and fixing that first one can make several others disappear on their own.

## "Missing Harness config" error

Exactly the error [Chapter 15](15-environment-variables-and-secrets.md)
and [Chapter 17](17-talking-to-the-harness-api.md) built on purpose —
it means one of `HARNESS_API_KEY`, `HARNESS_ACCOUNT_ID`, or
`HARNESS_BASE_URL` isn't set. Check:

- Does `.env.local` actually exist at your project's root (not inside
  `app/` or any other subfolder)?
- Did you restart your dev server after creating or editing `.env.local`?
  Next.js only reads environment variables when the server *starts* — a
  change to `.env.local` while `npm run dev` is already running won't
  take effect until you stop it (Ctrl+C in its terminal) and run
  `npm run dev` again.
- If you don't want to use a real Harness account at all, set
  `USE_MOCK_DATA=true` instead — see [Appendix A](appendix-a-mock-data-fallback.md).

## The filter bar's dropdowns never populate

This is a genuinely good moment to practice the debugging habit from
[Chapter 18](18-first-api-route.md): test the underlying API route
directly, bypassing the UI entirely. Open
`http://localhost:3000/api/orgs` directly in your browser. If *that*
shows an error or empty data, the problem is in your data layer
(`lib/harness.ts` or the route itself) — work backward from there. If
*that* works fine but the dropdown still doesn't populate, the problem is
specifically in the `FilterBar` component's fetch/state logic from
[Chapter 20](20-building-the-filter-bar.md) — check your browser's
developer console (usually opened with F12, or right-click → Inspect) for
any errors logged there.

## A hydration warning in the browser console

Read [Chapter 27](27-polish-accessibility-and-bugfixes.md)'s "Bug class 3"
section in full — this exact situation, why it happens, and how to fix it
(or confirm it's a harmless, expected one-time mismatch) is covered there
in detail, not just as a quick fix here.

## ESLint complains about something that seems fine

Run `npm run lint` directly and read its output carefully — ESLint
usually explains *why* a pattern is flagged, often with a rule name you
can search for if the explanation alone isn't enough. A few patterns this
book deliberately steers around, which you may see ESLint flag if you
write code differently than the book's examples: calling `setState`
directly inside a `useEffect` purely to keep one piece of state in sync
with another derivable value (see [Chapter 21](21-fetching-and-displaying-executions.md)'s
"clamp during render" discussion), and mutating a piece of state (an
array, object, or `Set`) in place instead of creating a new copy before
updating it (see [Chapter 21](21-fetching-and-displaying-executions.md)'s
discussion of `toggle` and immutable updates).

## The PDF export produces a blank or broken page

Check whether you clicked "Export" while the page was still showing a
loading message — [Chapter 26](26-exporting-a-pdf-report.md)'s
`waitForContentToSettle` function should prevent this, but if you're
still hitting it, confirm your loading message actually matches the exact
regular expression pattern that function checks for. If you've customized
any loading text, update `LOADING_TEXT_PATTERN` to match your new wording.

## Nothing in this appendix matches your problem

Slow down and isolate the smallest possible piece of the system that's
misbehaving:

1. If it involves data — test the specific API route directly with curl
   or your browser, with no UI involved at all (Chapter 18's technique).
2. If it involves a calculation — temporarily add a `console.log(...)`
   right before the suspicious line, printing the exact values going into
   it, and check your terminal (for server-side code, like API routes and
   `lib/` files) or your browser's developer console (for client
   components) for the output.
3. If it involves rendering — comment out everything except the smallest
   piece you suspect is broken, confirm *that* piece alone works, then add
   the rest back one piece at a time until it breaks again — at which
   point you've found exactly what caused it.

This process — isolate, inspect, narrow — is the same debugging mindset
[Chapter 27](27-polish-accessibility-and-bugfixes.md) modeled on four real
bugs from this exact project. It works on bugs this appendix never
anticipated too.
