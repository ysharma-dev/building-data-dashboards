# Chapter 18 — Your First API Route

**Skill:** understanding the full request path from a user's browser to a
third-party API and back — and why you almost always want your own small
backend sitting in between, rather than having the browser talk to the
outside API directly. This is the architectural decision that shapes
literally every dashboard you'll ever build on top of someone else's data.

## Why not just call Harness directly from the browser?

You might reasonably ask: we already wrote `listOrgs()` in
[Chapter 17](17-talking-to-the-harness-api.md) — why not just call it
directly from a React component running in the user's browser, and skip a
layer?

Two hard reasons, and one softer one:

1. **Your API key would be exposed.** Anything that runs in the browser is
   downloaded to and executed on the visitor's own computer — which means
   they (or anyone inspecting the page's network traffic) can see every
   value baked into that code, including any secret you tried to hide in
   it. `HARNESS_API_KEY` must **never** reach browser-side code. It can
   only ever be read on the server, where the visitor has no visibility
   into the running process.
2. **CORS.** Most APIs (Harness included) only allow requests from
   approved origins for security reasons — a browser calling Harness's API
   directly from `localhost:3000` would likely be blocked outright by a
   security mechanism called CORS (Cross-Origin Resource Sharing), which
   exists specifically to stop arbitrary websites from making authenticated
   requests to APIs on a visitor's behalf without permission.
3. **You gain a layer to reshape, cache, or combine data**, on your own
   terms, before it ever reaches the browser — useful even when the two
   reasons above don't apply.

The fix for all of this is exactly the Next.js feature previewed in
[Chapter 7](07-what-is-nextjs.md): a **Route Handler** — a file named
`route.ts` that runs *only* on the server, defines functions like `GET` or
`POST`, and becomes a URL your own frontend can call. The full request path
becomes:

```
Browser  →  Your Next.js API route (server)  →  Harness API (server)  →  back to browser
```

Your API key lives only in the middle step, on your own server — the
browser only ever talks to *your* routes, never to Harness directly.

## Building app/api/orgs/route.ts

Let's build the simplest possible slice of this end-to-end, right now: the
org-listing endpoint. Create a folder structure
`app/api/orgs/route.ts`. Recall from [Chapter 7](07-what-is-nextjs.md):
inside the `app/` folder, a file's *path* becomes its URL — so this file
will be reachable at `/api/orgs`.

```ts
import { NextResponse } from "next/server";
import { listOrgs } from "@/lib/harness";

export async function GET() {
  try {
    const orgs = await listOrgs();
    return NextResponse.json(orgs);
  } catch (err) {
    return NextResponse.json(
      { error: err instanceof Error ? err.message : "Unknown error" },
      { status: 500 },
    );
  }
}
```

Line by line:

- **`export async function GET()`** — this exact name, `GET`, is how
  Next.js recognizes this function as the handler for HTTP `GET` requests
  (the kind of request a browser makes by default — "give me data," as
  opposed to `POST`, which typically means "here's data, do something with
  it"). Export a `GET` function from a `route.ts` file, and Next.js
  automatically wires it up to respond at that file's URL path.
- **`await listOrgs()`** — this is the exact function you wrote in
  [Chapter 17](17-talking-to-the-harness-api.md). Notice how thin this file
  is: all the real work (talking to Harness, handling the mock-data flag,
  reshaping the response) already happened in `lib/harness.ts`. This route
  is nothing but a thin wrapper — call the function, and package up
  whatever it returns as an HTTP response.
- **`NextResponse.json(orgs)`** — a Next.js helper that turns a JavaScript
  value into a proper HTTP response with the right headers (`Content-Type:
  application/json`) and status code (`200 OK` by default), so the caller
  on the other end knows exactly how to parse what comes back.
- **The `try`/`catch` block** — recall from [Chapter 15](15-environment-variables-and-secrets.md)
  and [17](17-talking-to-the-harness-api.md) that `getConfig()` and
  `harnessFetch()` both `throw` an `Error` when something goes wrong
  (missing configuration, a failed request to Harness, and so on). If we
  didn't catch that here, an unhandled error inside a Route Handler would
  produce Next.js's generic, unhelpful "Internal Server Error" page. By
  catching it explicitly, we return a proper JSON error response instead —
  `{ error: "<the actual message>" }` — with an HTTP status of `500`
  (the standard code meaning "something went wrong on the server"). This
  gives whatever calls this route (a browser component, or you testing
  with curl) a real, readable error message to work with, rather than a
  mystery failure.
- **`err instanceof Error ? err.message : "Unknown error"`** — a small
  defensive detail: in JavaScript, it's technically possible (though rare)
  for something other than a proper `Error` object to be thrown. This check
  safely extracts `.message` only when we're confident it exists, falling
  back to a generic message otherwise, so this code never itself crashes
  while trying to report a *different* crash.

## Testing it before any UI exists

Here's a genuinely important habit: **you don't need a user interface to
test an API route.** With your dev server running (`npm run dev`, from
[Chapter 13](13-project-setup.md)), you can hit this endpoint directly.

**Option A — your browser.** Since `GET` is what a browser sends by
default when you visit a URL, just navigate to:

```
http://localhost:3000/api/orgs
```

You should see raw JSON printed in the browser window — an array of
objects, each with an `identifier` and `name` (or, if you left
`USE_MOCK_DATA=true` from [Chapter 15](15-environment-variables-and-secrets.md),
you'll see the mock organizations defined in
[Appendix A](appendix-a-mock-data-fallback.md)).

**Option B — curl**, a command-line tool for making HTTP requests, useful
because it shows you exactly what's happening without a browser's extra
formatting getting in the way. In a second terminal (leaving `npm run dev`
running in the first):

```bash
curl http://localhost:3000/api/orgs
```

This prints the same raw JSON directly in your terminal. If you deliberately
break something — say, comment out `HARNESS_API_KEY` in `.env.local` and
restart the dev server — try the request again and confirm you get back a
proper JSON error object with a `500` status, rather than a blank page or a
crash. Seeing your own error handling actually work, on purpose, before you
ever need it for real, is worth the thirty seconds it takes.

## Why testing an API route in isolation matters

This "test with curl before building any UI" habit is worth calling out
explicitly, because it's easy to skip as a beginner and skip a genuinely
useful debugging tool. When something eventually goes wrong later in this
book — a dropdown that won't populate, say — you'll want to be able to ask
"is the problem in my UI code, or is the problem in the data coming back
from my API route?" Being able to check the API route directly, with curl,
completely independent of any React component, answers that question in
seconds instead of leaving you guessing.

## Checkpoint

- [ ] `app/api/orgs/route.ts` exists and exports an async `GET` function.
- [ ] Visiting `http://localhost:3000/api/orgs` in your browser (with your
      dev server running) shows a JSON array.
- [ ] Running `curl http://localhost:3000/api/orgs` in a terminal shows the
      same JSON.
- [ ] Temporarily breaking your `.env.local` configuration and re-testing
      produces a JSON error response with a `500` status, not a crash or a
      blank page.

**This generalizes to:** any time your app needs data from a third-party
API, wrap that API call in your own thin Route Handler before your frontend
ever touches it — never call an authenticated third-party API directly from
browser code. And regardless of what that API is, get in the habit of
testing each new route by itself, with curl or your browser, the moment you
write it — before wiring up any UI to consume it. It's the fastest way to
tell "my data layer is broken" apart from "my UI is broken," and you'll be
glad you can tell the difference the first time something breaks two layers
deep.

Next: [Chapter 19 — The Remaining API Routes](19-remaining-api-routes.md)
