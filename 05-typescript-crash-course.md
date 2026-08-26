# Chapter 5 — TypeScript Crash Course

The dashboard app in this book is written in TypeScript, not plain
JavaScript — you'll see `.ts` and `.tsx` file extensions throughout Part 3,
starting with Chapter 13's project setup and becoming especially important
in Chapter 16, where you'll define the exact shape of every piece of data
the app works with. This chapter explains what TypeScript actually is and
teaches the small slice of it this project needs — just enough to
comfortably read code like `lib/types.ts` later in the book.

## What is TypeScript?

**TypeScript** is JavaScript with one thing added: a **type-checking
layer**. Every valid JavaScript program is already very close to being a
valid TypeScript program — TypeScript doesn't replace JavaScript or
introduce a different way of writing loops or functions. What it adds is
the ability to describe, in your code, exactly what *kind* of value
something is supposed to be — and then have a tool check, before your code
ever runs, whether you've actually respected those descriptions everywhere.

An important detail: browsers and Node.js only understand plain JavaScript
— they cannot run TypeScript directly. A separate step (called
**compiling**, or in this ecosystem, often just "type-checking" plus a
build tool) converts your `.ts`/`.tsx` files into plain `.js` before they
actually run. You won't need to manage this conversion by hand — Chapter 13
sets up a project where Next.js handles it automatically every time you
save a file or build the app. For now, the mental model is: **you write
`.ts`/`.tsx`, and something else quietly turns it into the plain JavaScript
that actually runs**, checking your types along the way.

## Why bother with types?

Consider this small, perfectly valid JavaScript function:

```js
function getStatusLabel(deployment) {
  return deployment.status.toUpperCase();
}
```

Nothing here tells you — or your code editor — what `deployment` is
actually supposed to look like. Does it have a `status` property? Is
`status` guaranteed to be text (so that `.toUpperCase()`, which only works
on strings, is safe to call)? In plain JavaScript, you find out the answer
by running the code and seeing if it crashes. If you pass in a deployment
object that happens not to have a `status` property, you get a runtime
error — but only once that specific code path actually runs, which might be
long after you wrote it, and possibly only when a real user hits it.

TypeScript lets you write down the answer *in advance*, and then checks
every place that function is used against that answer, before you ever run
the program:

```ts
type Deployment = {
  status: string;
};

function getStatusLabel(deployment: Deployment) {
  return deployment.status.toUpperCase();
}
```

Now, if anyone tries to call `getStatusLabel` with something that doesn't
have a `status` property (or has one that isn't text), TypeScript reports
an error immediately — usually right in your code editor, underlined in
red, before you even save the file, let alone run it.

This gives you three concrete, practical benefits that matter throughout
this book:

- **Catches mistakes early.** A typo in a property name (`deployment.staus`
  instead of `deployment.status`) is caught instantly, instead of surfacing
  as a confusing crash later.
- **Makes autocomplete smarter.** Because your editor knows the exact shape
  of `deployment`, it can suggest `status` (and any other properties) as
  you type, instead of you having to remember or go look them up.
- **Documents what a function expects**, directly in the code, in a form
  that's checked and enforced — not just a comment that can silently go out
  of date as the code changes around it.

## The `type` keyword: describing the shape of data

TypeScript's `type` keyword defines a named shape — a description of what
properties an object has, and what type each one is. This is the single
most important TypeScript feature for this book, because Chapter 16 is
built almost entirely around defining types like this for every kind of
data the dashboard app deals with.

```ts
type Deployment = {
  id: string;
  status: string;
  durationInSeconds: number;
  isSuccessful: boolean;
};
```

This says: "A `Deployment` is an object with an `id` (text), a `status`
(text), a `durationInSeconds` (a number), and an `isSuccessful` (a
boolean)." Any value you declare as a `Deployment` from here on will be
checked against exactly this shape.

```ts
const deployment: Deployment = {
  id: "exec-123",
  status: "Success",
  durationInSeconds: 42,
  isSuccessful: true,
};
```

The `: Deployment` after `deployment` is called a **type annotation** — it
tells TypeScript "this variable must match the `Deployment` shape," and
TypeScript will immediately flag it as an error if a required property is
missing, misspelled, or holds the wrong kind of value.

## Optional fields with `?`

Sometimes a property genuinely might not always be present. Putting a `?`
right after a property's name marks it as **optional** — allowed to be
present or absent, both are valid.

```ts
type Deployment = {
  id: string;
  status: string;
  errorMessage?: string;
};
```

Here, `errorMessage` is optional — a successful deployment might simply not
have one. Both of the following are valid `Deployment` values:

```ts
const success: Deployment = { id: "exec-1", status: "Success" };
const failure: Deployment = {
  id: "exec-2",
  status: "Failed",
  errorMessage: "Timed out waiting for approval",
};
```

Marking a field optional does more than just permit skipping it — it also
forces you (via a TypeScript error, if you forget) to handle the "it might
not be there" case anywhere you try to use it, which prevents a whole class
of bugs where code assumes a value exists and crashes when it doesn't.

## Union types: `string | null`

A **union type** describes a value that could be *one of several specific
types*, using the `|` symbol (read as "or") between them.

The most common union you'll see throughout this book's later chapters is
`string | null` — describing a value that is either some text, or the
special value `null`, which represents "intentionally no value."

```ts
type Filters = {
  selectedOrg: string | null;
};
```

This says: "`selectedOrg` is either a string, or explicitly nothing has
been selected yet." This comes up constantly in the dashboard's filter bar
(Chapter 20) — before the user picks an organization from a dropdown, there
simply isn't a value yet, and `string | null` lets the code represent "no
selection" honestly, rather than pretending an empty string or a made-up
placeholder text means the same thing.

Union types aren't limited to two options, and aren't limited to mixing
with `null` — you'll also see them used to restrict a value to one of a
specific fixed set of text options, which is a pattern worth recognizing:

```ts
type DeploymentStatus = "Success" | "Failed" | "Running";
```

This says a `DeploymentStatus` isn't just *any* string — it must be
exactly one of these three specific pieces of text. Assigning anything else
(including a typo like `"Sucess"`) is a TypeScript error, caught before the
code ever runs.

## How `.ts`/`.tsx` files relate to plain JavaScript

Two file extensions matter for this book:

- **`.ts`** — a plain TypeScript file, used for code with no visual UI in
  it — type definitions (Chapter 16), functions that fetch or process data
  (Chapters 17, 22), and API route handlers (Chapter 18).
- **`.tsx`** — TypeScript combined with **JSX**, the HTML-like syntax
  you'll learn about in [Chapter 6](06-what-is-react.md) for describing
  what appears on screen. Any file that defines a React component (a
  reusable piece of UI — again, Chapter 6 covers this properly) uses `.tsx`.

You don't need to decide which extension to use very often yourself — the
project structure set up in Chapter 13 will make it obvious which files are
which, and your code editor will complain clearly if you write JSX inside a
plain `.ts` file by mistake.

Under the hood, both extensions are converted into plain `.js` before they
run, exactly as described earlier in this chapter — the type annotations
(`: Deployment`, `?`, `| null`, and so on) are stripped out entirely during
that conversion. This is worth understanding clearly: **types exist purely
to help you while you're writing and checking your code — they have zero
effect on how the program actually behaves once it's running.** They're a
tool for catching mistakes early, not a feature of the running app itself.

## Checkpoint

- You can explain, in one sentence, what TypeScript adds on top of
  JavaScript.
- Given a `type` definition with a few properties, you could write a valid
  object matching it.
- You can explain what a `?` after a property name means, and why
  `string | null` is more honest than using an empty string to mean "no
  selection yet."
- You understand that type annotations are removed before the code actually
  runs — they only affect the checking step, not the running program.

Next: [Chapter 6 — What Is React?](06-what-is-react.md)
