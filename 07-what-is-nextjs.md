---
layout: chapter
title: "Chapter 7 — What Is Next.js?"
nav_order: 8
permalink: /07-what-is-nextjs/
---

# Chapter 7 — What Is Next.js?

React (Chapter 6) gives you components, but on its own, React doesn't
decide which component shows up at which web address, and it doesn't have
an opinion about where your code runs — your browser or a server somewhere.
**Next.js** is a framework built on top of React that answers both of those
questions, plus several more, and it's the framework this book's entire
project — Harness Deploy Insights — is built with, starting in Chapter 13.
This chapter explains what Next.js actually adds, since almost every
structural decision in Part 3 (why a file lives where it does, why some
files start with `"use client"` and others don't) traces back to the ideas
in this chapter.

A **framework**, in this context, is a structured foundation that makes a
lot of decisions for you upfront — where files go, how a page becomes
visible on the web, how your own backend code is organized — so that you
don't have to invent those decisions yourself for every new project.

## File-based routing: a file's location becomes its URL

In many older or more manual web setups, you have to write explicit code
that says "when someone visits `/pipelines`, show this component." Next.js
takes a different, more direct approach called **file-based routing**: the
*location* of a file inside a special folder named `app/` directly becomes
the URL path that shows it — no separate routing code required.

For example, a file at:

```bash
app/pipelines/page.tsx
```

automatically becomes visible at the URL path `/pipelines`, with zero extra
configuration. A file at `app/page.tsx` (with nothing in between) is the
site's home page, visible at `/`. Each folder inside `app/` becomes one
segment of the URL, and a file literally named `page.tsx` inside that
folder is what actually renders for that URL.

This might seem like a small convenience, but it removes an entire category
of bookkeeping: you never have to maintain a separate list mapping URLs to
components — the folder structure *is* that list, visibly, in your project.
When Chapter 13 sets up this book's project, the `app/` folder's structure
will already tell you a lot about what pages the app has, just by looking
at it.

## The App Router (and why you might hear about a "Pages Router")

The specific routing system just described — based on an `app/` folder — is
called the **App Router**, and it's the current, recommended way to build a
Next.js project. It's the only routing system this book uses.

If you search for Next.js tutorials online, though, you will likely run
into older material that instead uses a folder named `pages/` (instead of
`app/`), where each file directly becomes a page without needing a
`page.tsx` filename convention. This older system is called the **Pages
Router**, and it was how every Next.js project worked before the App
Router existed. It still works and is still supported, which is exactly
why you'll keep running into it in older tutorials, blog posts, and Stack
Overflow answers.

You don't need to learn the Pages Router to follow this book — just
recognize the name if you see it, and know that a `pages/` folder in a
tutorial means you're looking at the *older* approach, with somewhat
different rules than everything in this book's `app/`-based project.

## Server Components and Client Components

This is the single most important — and most initially confusing — idea
Next.js adds on top of plain React, so it's worth slowing down here.

Plain React (Chapter 6) doesn't inherently care whether your component's
code runs in the user's browser or somewhere else — historically, it was
almost always assumed to run entirely in the browser. Next.js introduces a
real choice: any component inside the `app/` folder can run in one of two
places.

**Server Components** run on the server — a computer elsewhere, not the
visitor's own browser — and this is the **default** for every file inside
`app/` unless you say otherwise. Because they run on a server rather than
in a stranger's browser, Server Components can safely do things you'd never
want happening in a browser: talking directly to a database, using a secret
API key, or reading a file from disk. Their output (the actual HTML) is
what gets sent to the browser — by the time a user's browser receives it,
it's just the finished result, with none of the server-side code or secrets
exposed.

**Client Components** run in the browser, exactly like the plain React
you learned in Chapter 6. You mark a file as a Client Component by putting
a special line, `"use client";`, at the very top of the file, before
anything else:

```tsx
"use client";

import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

Client Components are required for anything **interactive** — anything
using `useState`, `useEffect`, or responding to events like `onClick`,
because all of that only makes sense running live in a user's browser,
reacting to what that specific user does. A Server Component, running once
on a server before anything reaches the browser, can't respond to a button
click that hasn't happened yet on some visitor's machine.

A simple rule of thumb, one you'll apply constantly in Part 3: **start by
assuming a component is a Server Component (the default), and only add
`"use client"` once you actually need interactivity or a React hook like
`useState`/`useEffect`.** The dashboard's filter bar (Chapter 20), with its
clickable dropdowns, will be a Client Component. A component that simply
displays a static layout around it might not need to be.

## API Route Handlers: your own backend, defined by a file

The last major piece Next.js adds is a way to define your own backend
endpoints — code that runs on the server and responds to a request, without
needing a separate backend project or server at all.

You do this by creating a file specifically named `route.ts`, inside the
`app/api/` folder. The file's location works exactly like page routing
(same idea, different folder): a file at

```bash
app/api/deployments/route.ts
```

becomes a backend endpoint reachable at the URL path `/api/deployments`.
Inside that file, you export a function named after the type of request it
should handle — most commonly `GET` (for fetching data) or `POST` (for
sending data):

```ts
export async function GET() {
  return Response.json({ message: "Hello from the server" });
}
```

Visiting `/api/deployments` in a browser (or having your own front-end code
`fetch()` that path, exactly as shown in Chapter 4's `async`/`await`
example) runs this `GET` function on the server and returns whatever it
responds with — here, a small JSON object.

### Why have your own backend routes in front of another API at all?

You might reasonably wonder: if the app ultimately needs data from some
outside service, why not just have the browser talk to that outside service
directly, and skip writing your own API routes entirely?

There are a few general reasons a real project almost always wants its own
backend layer sitting in between, even when a perfectly good outside API
already exists:

- **Keeping secrets safe.** Talking to most real APIs requires an API key
  (a secret credential, like a password). If your browser-side code talked
  to that API directly, the key would have to be embedded somewhere the
  browser can read it — which means anyone visiting your site could find
  and steal it. A Server Component or an API route, running only on the
  server, can hold that secret and use it on the app's behalf, without ever
  sending it to the browser.
- **Reshaping the data.** An outside API often returns far more data, or a
  differently-shaped structure, than your app actually needs. Your own API
  route can call the outside API, then trim and reshape the response into
  exactly the shape your front-end components expect.
- **Combining multiple sources.** A single API route can call several
  outside services (or the same one several times) and combine the results
  into one clean response for your app.

This exact pattern — your own `route.ts` file sitting in front of an
outside API — is precisely what [Chapter 18](18-first-api-route.md) builds,
once you've met the actual outside API (Harness) it's protecting, in
[Chapter 10](10-harness-and-cd-pipelines.md). For now, the mechanism is the
part to hold onto: **a file named `route.ts`, with an exported `GET` or
`POST` function, becomes a backend endpoint at that file's URL path** — no
separate server, no separate project, just another file inside `app/`.

## Checkpoint

- You can explain, in one sentence, what file-based routing means.
- You know that this book only uses the App Router, and that a `pages/`
  folder in an outside tutorial signals the older Pages Router instead.
- You can explain the difference between a Server Component (default, runs
  on the server, no `useState`/`onClick`) and a Client Component (marked
  with `"use client"`, runs in the browser, required for interactivity).
- You can explain what a file named `route.ts` under `app/api/` does, and
  name one general reason a project wants its own backend routes rather
  than calling an outside API directly from the browser.

Next: [Chapter 8 — What Is Tailwind CSS?](08-what-is-tailwind.md)
